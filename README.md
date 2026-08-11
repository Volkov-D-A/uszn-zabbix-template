# uszn-zabbix-template

Учебная сборка собственного шаблона Zabbix для Astra Linux.

## Шаг 01. Версия сборки Astra Linux

Первый элемент данных читает файл:

```text
/etc/astra_version
```

Ключ Zabbix agent:

```text
vfs.file.contents[/etc/astra_version]
```

## Шаг 02. Версия ядра Linux

Версия текущего ядра получается штатным ключом Zabbix agent:

```text
system.sw.os[full]
```

Предобработка извлекает из `/proc/version` только release ядра.
При его изменении Zabbix создаёт информационное событие.

Минимальный шаблон:

```text
templates/astra_linux_basic.yaml
```

## Документация

Текущее состояние шаблона:

```text
docs/current-template-state.md
```
