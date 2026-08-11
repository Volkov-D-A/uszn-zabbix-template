# Мониторинг auditd и PARSEC

Этот документ фиксирует предложения по развитию security-мониторинга Astra Linux
на основе журнала auditd и предустановленных шаблонов Zabbix:

```text
example/zabbix_astra_template.xml
example/zabbix_astra_parsec_template.xml
```

## Принципы

Решение должно сохранять принятые для проекта ограничения:

- Zabbix 7.0 и `zabbix-agent2`;
- активные проверки агента;
- без `system.run` и `UserParameter`;
- сбор прежде всего security-событий, а не всего журнала;
- Zabbix используется для счетчиков, порогов и оперативных оповещений, но не
  заменяет SIEM для полноценной корреляции событий.

## Что содержат предустановленные шаблоны

### Общий шаблон auditd

`zabbix_astra_template.xml` содержит 13 активных `logrt[]` items:

```text
ANOM
CAPS
CONFIG
CRYPTO
EXECVE
GROUP
GRP
KERNEL
LOGIN
RESP
SYSTEM
USER
VIRT
```

Все элементы читают `/var/log/audit/audit.log` и сохраняют совпавшие строки как
`LOG`.

### Шаблон PARSEC

`zabbix_astra_parsec_template.xml` содержит два фильтра:

```text
parsec-f
parsec-p
```

`parsec-f` является ключом правил файлового PARSEC-аудита. `parsec-p` является
ключом правил аудита процессов, пользователей и сетевых операций. Эти ключи
назначаются правилам Linux Audit параметром `-k`.

Документация Astra Linux:

```text
https://astra.ru/local/ajax/download/docs/12/21537/
```

## Почему шаблоны не следует копировать целиком

В примерах обнаружены следующие ограничения:

- нет триггеров;
- все совпавшие строки хранятся 90 дней;
- фильтры слишком широкие и пересекаются;
- не задан `mode=skip`, поэтому новый item может начать со старых записей;
- ротация файлов не описана явным регулярным выражением;
- имена, UUID и теги не соответствуют нашему шаблону.

Примеры пересечений и потерь:

- `USER_LOGIN` совпадает одновременно с фильтрами `USER` и `LOGIN`;
- `CRYPTO_*_USER` совпадает с `CRYPTO` и `USER`;
- общий `EXECVE` способен создать большой объем истории и сохранить аргументы
  запуска программ;
- описание `ANOM` упоминает `INTEGRITY_DATA`, но фильтр `ANOM_` его не найдет.

Следовательно, из шаблонов нужно заимствовать назначение категорий и PARSEC-ключи,
но создать собственные items, регулярные выражения, триггеры, теги и UUIDv4.

## Ограничение формата auditd

Одно audit-событие может состоять из нескольких строк с одинаковыми timestamp и
serial number в поле `msg=audit(...)`. `logrt[]` и `logrt.count[]` обрабатывают
отдельные строки и не объединяют их в полное событие.

Это означает, что Zabbix хорошо подходит для:

- подсчета определенных типов записей;
- пороговых триггеров;
- доставки выбранных критичных строк;
- оперативных оповещений.

Для корреляции всех записей одного события, длительного расследования и сложных
правил необходима SIEM или специализированный обработчик Linux Audit.

## Рекомендуемые security-категории

### Неудачная аутентификация

```regex
type=(USER_AUTH|USER_LOGIN).*res=failed
```

Рекомендуется `logrt.count[]` с порогами, вынесенными в макросы:

```text
5 событий за 5 минут  -> WARNING
20 событий за 5 минут -> HIGH
```

### Изменения учетных записей и групп

```regex
type=(ADD_USER|DEL_USER|ADD_GROUP|DEL_GROUP|USER_CHAUTHTOK|USER_MGMT|GRP_MGMT|ROLE_ASSIGN|ROLE_REMOVE)
```

Любое событие должно создавать `INFO` или `WARNING` в зависимости от политики
уведомлений.

### Изменения security-конфигурации

```regex
type=(USYS_CONFIG|DAEMON_CONFIG|CONFIG_CHANGE|MAC_CONFIG_CHANGE|USER_MAC_CONFIG_CHANGE)
```

Любое событие — `WARNING`. Отключение или критическая ошибка аудита — `HIGH`.

### Ошибки auditd

```regex
type=(DAEMON_ABORT|DAEMON_ERR)
```

Любое событие — `HIGH`.

### Запреты мандатного доступа

Базовый вариант:

```regex
type=(AVC|USER_AVC).*denied
```

Любое событие — `HIGH`. Фильтр необходимо проверить на реальных строках Astra
Linux: формат PARSEC-событий может отличаться от SELinux AVC.

### Аномалии и нарушения целостности

```regex
type=(ANOM_[A-Z_]+|INTEGRITY_[A-Z_]+)
```

Любое событие — `HIGH`.

### Реакции системы безопасности

```regex
type=RESP_[A-Z_]+
```

Эти события редки и важны. Для них целесообразно сохранить исходную строку и
создать проблему severity `HIGH`.

### Критичные audit keys

Если такие метки реально используются в `/etc/audit/rules.d/`, можно отдельно
отслеживать:

```text
key="identity"
key="auditconfig"
key="time-change"
key="modules"
key="privileged"
```

Набор ключей должен соответствовать фактической конфигурации auditd на клиентах.

### PARSEC-аудит

```regex
key="parsec-f"
key="parsec-p"
```

`parsec-f` относится к файловому аудиту, а `parsec-p` — к аудиту процессов и
пользователей. Оба фильтра могут быть высоконагруженными. На первом этапе лучше
собирать количество событий, а сырые строки сохранять только для узкого набора
критичных операций.

## Стратегия сбора

### Счетчики

`logrt.count[]` рекомендуется использовать для:

- неудачной аутентификации;
- массовых и высоконагруженных категорий;
- `parsec-f` и `parsec-p`;
- порогов за заданный интервал времени.

Пример:

```text
logrt.count["/var/log/audit/^audit\.log(\.[0-9]+)?$","type=(USER_AUTH|USER_LOGIN).*res=failed","UTF-8",1000,skip]
```

### Исходные строки

`logrt[]` с типом данных `LOG` имеет смысл оставить только для:

- ошибок auditd;
- реакций `RESP_*`;
- аномалий и нарушений целостности;
- изменений security-конфигурации;
- наиболее важных отказов мандатного доступа.

Не следует передавать в Zabbix весь `audit.log`: он может содержать имена
пользователей, команды, пути и другие чувствительные данные. Срок хранения сырых
строк должен быть заметно меньше 90 дней и определяться после измерения объема.

## Параметры log items

Базовые параметры новых items:

```text
type: ZABBIX_ACTIVE
delay: 30s или 1m
mode: skip
encoding: UTF-8
maxproclines: подобрать нагрузочным тестом
```

Регулярное выражение для основного файла и типичной числовой ротации:

```regex
/var/log/audit/^audit\.log(\.[0-9]+)?$
```

Фактическую схему имен ротации нужно проверить на Astra Linux. `mode=skip`
предотвращает первичную обработку старого журнала для нового item. Для
security-событий нежелательно задавать `maxdelay`, способный отбросить старые
строки при отставании.

Документация Zabbix:

```text
https://www.zabbix.com/documentation/7.0/en/manual/config/items/itemtypes/zabbix_agent
https://www.zabbix.com/documentation/7.0/en/manual/config/items/itemtypes/zabbix_agent/log_items
```

## Права доступа

`zabbix-agent2` должен иметь право чтения `/var/log/audit/audit.log`. Без него
log item перейдет в `NOTSUPPORTED`.

Варианты организации доступа:

1. Строго ограниченный read-доступ пользователю `zabbix` с учетом ротации.
2. Вывод выбранных audit-событий в отдельный файл, доступный группе `zabbix`.
3. Передача полного аудита в SIEM, а в Zabbix — только счетчиков и критичных
   оповещений.

Не следует давать агенту избыточные привилегии или доступ ко всем системным
журналам. В Astra Linux SE нужно учитывать одновременно Unix-права и мандатную
политику доступа.

## Предлагаемая структура шаблонов

Вместо помещения всех log items в `Template Astra Linux Basic` предлагается
создать два компонентных шаблона:

```text
Template Astra Linux Audit
Template Astra Linux PARSEC
```

`Template Astra Linux Audit` применяется ко всем Astra-клиентам с auditd.
`Template Astra Linux PARSEC` применяется только к Astra Linux SE с активными
PARSEC-правилами.

Для новых объектов необходимо:

- создавать новые UUIDv4;
- сохранить теги `scope`, `component`, `control`;
- использовать `component=audit` и `component=parsec`;
- добавить `control=event` для security-событий;
- вынести пороги и интервалы в макросы;
- не копировать UUID, группы и имена из XML-примеров.

## Первая итерация

Минимальный полезный набор:

```text
Audit: failed authentication count
Audit: security configuration changes
Audit: audit daemon errors
Audit: account and group changes
Audit: anomalies and integrity violations
Audit: mandatory access denials
PARSEC: file audit events count
PARSEC: process audit events count
```

Перед реализацией необходимо проверить:

1. Реальные строки `audit.log` на Astra Linux 1.8.
2. Формат имен ротированных файлов.
3. Доступ пользователя `zabbix` к журналу.
4. Фактические правила и ключи в `/etc/audit/rules.d/`.
5. Объем `EXECVE`, `parsec-f` и `parsec-p` за рабочий день.
6. Регулярные выражения через тест предобработки Zabbix.
7. Нагрузку на агент, сеть, Zabbix server/proxy и базу данных.

## Дальнейшее развитие

После первой итерации можно добавить:

- разные пороги severity через макросы;
- контекстные макросы для разных групп хостов;
- отдельные фильтры для критичных audit keys;
- dashboard по категориям `authentication`, `accounts`, `configuration`,
  `integrity`, `parsec`;
- маршрутизацию уведомлений по тегам;
- передачу полного аудита в SIEM при сохранении в Zabbix только счетчиков и
  критичных оповещений.
