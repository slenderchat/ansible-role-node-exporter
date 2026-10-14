# ansible-role-node-exporter

[English](README.md) | [Русский](README.ru.md)

Ansible-роль для установки и настройки [Prometheus Node Exporter](https://github.com/prometheus/node_exporter) на РЕД ОС 7.3 и 8 из системных RPM-пакетов с использованием штатных unit-файлов systemd.

Автор реализации — Nikita Grabar.

> **Авторские права.** Copyright (c) 2026 Nikita Grabar. All rights reserved. Проект не распространяется под открытой лицензией. Использование, копирование, изменение и распространение требуют предварительного разрешения правообладателя. См. [COPYRIGHT](COPYRIGHT).

## Поддерживаемые платформы

- РЕД ОС 7.3 (проверена версия Node Exporter 1.10.2).
- РЕД ОС 8 (проверена версия Node Exporter 1.12.1).

Развёртывание проверено на обеих ОС, на узлах как с уже установленным пакетом, так и без него. Подтверждены первоначальный запуск, идемпотентность (`changed=0` при повторении), check mode и доступность `/metrics`.

## Требования

- Ansible 2.10 или новее на управляющем узле (разработка выполнялась с ansible-core 2.19).
- Доступность RPM-пакета `prometheus-node_exporter` в репозиториях целевой ОС.
- Systemd и поставляемые пакетом unit-файлы `node_exporter.service` и `node_exporter.socket`.
- Свободный TCP-порт `9100`: работающий Docker-контейнер или другой экземпляр экспортера должен быть остановлен перед запуском роли.
- Повышение привилегий (`become: true`) для установки пакета и управления настройками сервиса.

## Установка

Разместите содержимое репозитория в каталоге `node_exporter` одного из путей поиска Ansible-ролей:

```bash
mkdir -p roles
git clone https://github.com/slenderchat/ansible-role-node-exporter.git roles/node_exporter
```

При необходимости укажите родительский каталог через `roles_path` или `ANSIBLE_ROLES_PATH`. Роль не зависит от других ролей.

## Конфигурация

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `node_exporter_ntp_server` | `127.0.0.1` | NTP-сервер, опрашиваемый коллектором NTP. |

Настройка `127.0.0.1` работает только при наличии локального NTP-сервера, принимающего запросы (например, соответствующим образом настроенного chronyd). Иначе `node_scrape_collector_success{collector="ntp"} 0`, но остальные коллекторы продолжают работать. При необходимости переопределите переменную в переменных инвентаря.

Остальные аргументы экспортера находятся в `templates/etc_sysconfig_node_exporter.j2`. Шаблон включает набор коллекторов: textfile, NTP, процессы, файловые системы, сеть, память, CPU, systemd и другие. Правила фильтрации и подробные параметры см. непосредственно в шаблоне. Каталог textfile-коллектора из пакета: `/var/lib/node_exporter/textfile_collector`.

### Экранирование аргументов

Шаблон записывает аргументы в `OPTIONS` файла `/etc/sysconfig/node_exporter`. Systemd сначала разбирает значение `EnvironmentFile`, а затем разделяет `$OPTIONS` в `ExecStart` на аргументы. Поэтому внутри значений аргументов недопустимы пробелы, кавычки и обратная косая черта. В регулярных выражениях используйте `[.]` вместо `\.`; пример находится в шаблоне.

## Использование

Создайте пользовательский playbook `site.yml` вне этого репозитория:

```yaml
---
- name: Deploy Node Exporter
  hosts: all
  become: true
  gather_facts: false
  roles:
    - node_exporter
```

Сначала проверьте изменения, затем выполните развёртывание на нужном узле:

```bash
ANSIBLE_ROLES_PATH=./roles ansible-playbook -i inventory.ini site.yml --limit HOST --check --diff
ANSIBLE_ROLES_PATH=./roles ansible-playbook -i inventory.ini site.yml --limit HOST --diff
```

Поскольку playbook использует `hosts: all`, для поэтапного развёртывания рекомендуется `--limit`.

### Действия роли

1. Устанавливает `prometheus-node_exporter` через `state: present` без фиксации версии и автоматического обновления существующего пакета.
2. Снимает маскирование `node_exporter.socket` и `node_exporter.service`, включает их автозапуск и запускает.
3. Записывает аргументы коллекторов в `OPTIONS` файла `/etc/sysconfig/node_exporter`.
4. Создаёт `/etc/systemd/system/node_exporter.service.d/restart.conf` с `Restart=on-failure`, не изменяя unit-файлы из RPM.
5. Перезагружает конфигурацию systemd при изменении drop-in и перезапускает сервис при изменении аргументов или drop-in. Без изменений лишнего restart нет.
6. Проверяет HTTP 200 по адресу `http://127.0.0.1:9100/metrics` при обычном запуске.

Порт `9100` задаётся штатным `node_exporter.socket` (`ListenStream=9100`). Экспортер использует socket activation (`--web.systemd-socket`); **не добавляйте** `--web.listen-address` в `OPTIONS`. Сервис работает от непривилегированного системного пользователя `node_exporter`, созданного пакетом.

## Проверка работоспособности

После развёртывания проверьте состояние сервиса, сокета и HTTP:

```bash
systemctl is-enabled node_exporter.socket node_exporter.service
systemctl is-active node_exporter.socket node_exporter.service
systemctl status node_exporter.service --no-pager
curl --noproxy '*' -fsS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:9100/metrics
```

При повторном выполнении Ansible ожидается `changed=0`. Режим `--check --diff` поддерживает узлы без RPM: если установка пакета только моделируется, запуск и автозапуск unit-файлов, а также restart пропускаются. HTTP-проверка в check mode пропускается всегда.

При миграции с Docker перед запуском штатного сокета остановите контейнер и освободите порт `9100`. Роль не управляет Docker и не реализует откат контейнерного развёртывания.

## Известные ограничения

- Часть коллекторов, в том числе `slabinfo` и RAPL energy, может не иметь прав для чтения данных от непривилегированного пользователя. При этом `node_scrape_collector_success 0`, но экспортер продолжает работать.
- Для NTP-коллектора нужен доступный NTP-сервер. Его ошибка не мешает работе других коллекторов.
- `Restart=on-failure` настроен, но автоматическое восстановление после искусственно вызванного завершения процесса отдельно не тестировалось.
- HTTP 200 не гарантирует успешность каждого коллектора: проверяйте `node_scrape_collector_success`.

## Авторские права

Copyright (c) 2026 Nikita Grabar. All rights reserved. Разрешение на использование не предоставляется автоматически; см. [COPYRIGHT](COPYRIGHT).
