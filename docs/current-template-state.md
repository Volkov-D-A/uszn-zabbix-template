# Текущее состояние шаблона Astra Linux

Файл шаблона:

```text
templates/astra_linux_basic.yaml
```

Целевая версия Zabbix:

```text
7.0
```

Целевая ОС:

```text
Astra Linux 1.8, Debian based
```

Агенты:

```text
zabbix-agent2
```

Шаблон ориентирован на активные проверки агента:

```text
type: ZABBIX_ACTIVE
```

## Группы

Группа шаблонов:

```text
Astra clients
```

Группа хостов:

```text
Astra clients
```

## Что уже мониторится

### Версия Astra Linux

Item:

```text
Astra Linux: build version
```

Key:

```text
vfs.file.contents[/etc/astra_version]
```

Назначение:

```text
Получить точную версию сборки Astra Linux, например 1.8.5.
```

Trigger:

```text
Astra Linux: build version has changed
```

Severity:

```text
INFO
```

Особенность:

```text
manual_close: YES
```

Триггер фиксирует факт изменения версии ОС. Это аудиторское событие, а не авария.

### Dr.Web: основная служба

Item:

```text
Dr.Web: config daemon active state
```

Key:

```text
systemd.unit.info[drweb-configd.service,ActiveState]
```

Ожидаемое значение:

```text
active
```

Trigger:

```text
Dr.Web: config daemon is not active
```

Severity:

```text
HIGH
```

### Dr.Web: актуальность баз

Item:

```text
Dr.Web: today database file modification time
```

Key:

```text
vfs.file.time[/var/opt/drweb.com/bases/dwrtoday.vdb,modify]
```

Назначение:

```text
Проверять время изменения текущего файла баз Dr.Web.
```

Trigger:

```text
Dr.Web: today database file was not updated recently
```

Severity:

```text
WARNING
```

Макросы:

```text
{$DRWEB.BASES.MAXAGE}=2d
{$DRWEB.BASES.NODATA}=2h
```

Логика:

```text
Если файл dwrtoday.vdb старше 2 дней и данные по item продолжают поступать,
создается предупреждение.
```

Важно:

```text
Ранее рассматривался файл /var/opt/drweb.com/bases/timestamp, но он не всегда
совпадает с временем создания/обновления dwrtoday.vdb. Поэтому выбран mtime
файла dwrtoday.vdb.
```

### Служба системного аудита

Item:

```text
Audit: auditd active state
```

Key:

```text
systemd.unit.info[auditd.service,ActiveState]
```

Ожидаемое значение:

```text
active
```

Trigger:

```text
Audit: auditd service is not active
```

Severity:

```text
HIGH
```

### Синхронизация времени

Item:

```text
Time sync: systemd-timesyncd active state
```

Key:

```text
systemd.unit.info[systemd-timesyncd.service,ActiveState]
```

Ожидаемое значение:

```text
active
```

Trigger:

```text
Time sync: systemd-timesyncd service is not active
```

Severity:

```text
WARNING
```

### Автоматизация обновлений Astra Linux

Item:

```text
Updates: astra-update-service active state
```

Key:

```text
systemd.unit.info[astra-update-service.service,ActiveState]
```

Ожидаемое значение:

```text
active
```

Trigger:

```text
Updates: astra-update-service is not active
```

Severity:

```text
WARNING
```

## Теги

Используются теги:

```text
scope
component
control
```

Базовый тег области:

```text
scope=security
```

Компоненты:

```text
component=os
component=drweb
component=audit
component=time
component=updates
```

Типы контроля:

```text
control=inventory
control=availability
control=freshness
```

Назначение тегов:

```text
Фильтрация Problems, построение dashboard widgets и будущая маршрутизация
уведомлений по компонентам безопасности.
```

## Что сознательно не используем

### UserParameter

От UserParameter отказались.

Причина:

```text
Нужны дополнительные конфиги агента и sudoers на большом количестве ПК.
```

Текущий принцип:

```text
Использовать только штатные ключи Zabbix agent2 и simple file/systemd checks.
```

### system.run

Не используем.

Причина:

```text
Для шаблона безопасности нежелательно включать удаленный запуск команд через
агент.
```

### agent.ping и icmpping в этом шаблоне

Пробовали добавлять:

```text
agent.ping
icmpping
```

Отказались.

Причина:

```text
После добавления менялась индикация доступности возле интерфейса хоста в Zabbix.
Доступность агента лучше оставить базовым инфраструктурным шаблонам, а этот
шаблон держать focused на security-состоянии Astra Linux.
```

## Текущие незакоммиченные изменения

На момент написания этого документа в шаблоне еще не закоммичены:

```text
теги item/trigger;
auditd.service;
systemd-timesyncd.service;
astra-update-service.service.
```

Перед продолжением в другом контексте стоит проверить:

```bash
git status --short
git diff -- templates/astra_linux_basic.yaml
```

## Возможные следующие шаги

1. Импортировать текущий шаблон в Zabbix и проверить Latest data.
2. Подтвердить реальные состояния служб на тестовом ПК.
3. Закоммитить текущий набор изменений.
4. Перейти к журналированию:

```text
systemd-journald
rsyslog, если используется
```

5. Позже рассмотреть контроль критичных файлов:

```text
/etc/passwd
/etc/group
/etc/sudoers
/etc/ssh/sshd_config
/etc/audit/audit.rules
```
