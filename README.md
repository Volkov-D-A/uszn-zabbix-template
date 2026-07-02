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

Минимальный шаблон:

```text
templates/astra_linux_basic.yaml
```
