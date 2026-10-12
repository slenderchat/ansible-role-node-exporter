# ansible-role-node-exporter

Ansible role for installing and configuring Prometheus Node Exporter on RED OS 7.3 and 8 using native packages and systemd.

## What it does

- Installs `prometheus-node_exporter` from the configured OS repositories (`state: present`: no version pinning, no automatic upgrades).
- Unmasks, enables and starts the packaged `node_exporter.socket` and `node_exporter.service` units.
- Writes exporter arguments to `OPTIONS` in `/etc/sysconfig/node_exporter`.
- Adds a systemd drop-in with `Restart=on-failure` without modifying the packaged units.
- Restarts the service only when its configuration changes, and reloads systemd only when the drop-in changes.
- Checks that metrics are served at `http://127.0.0.1:9100/metrics`.

Port 9100 comes from the packaged socket unit (systemd socket activation). The exporter runs as the unprivileged `node_exporter` user.

Detailed role documentation (in Russian) is in [`roles/node_exporter/README.md`](roles/node_exporter/README.md).

## Requirements

- Control node: Ansible 2.10 or later (developed with ansible-core 2.19).
- Managed hosts: RED OS 7.3 or RED OS 8 with the `prometheus-node_exporter` package available in the configured repositories.
- Port 9100 must be free on the managed hosts.
- Privilege escalation (`become`) on the managed hosts.

## Installation

```bash
git clone <repository-url> ansible-role-node-exporter
cd ansible-role-node-exporter
```

The playbook refers to the role by name, so point Ansible at the `roles` directory, for example with `ANSIBLE_ROLES_PATH=./roles`, or with `roles_path = ./roles` in a local `ansible.cfg`.

## Usage

Create an inventory, for example `inventory.ini` (ignored by Git):

```ini
[node_exporter]
host1.example.com
host2.example.com
```

Run the playbook:

```bash
ANSIBLE_ROLES_PATH=./roles ansible-playbook -i inventory.ini playbooks/node_exporter.yml
```

The playbook targets `hosts: all`; use `--limit` to restrict it.

Preview the changes without applying them:

```bash
ANSIBLE_ROLES_PATH=./roles ansible-playbook -i inventory.ini playbooks/node_exporter.yml --check --diff
```

In check mode the `/metrics` check is skipped. If the package is not yet installed, unit enable/start and the service restart are skipped too, because the units do not exist yet.

## Variables

| Variable | Default | Description |
|---|---|---|
| `node_exporter_ntp_server` | `127.0.0.1` | NTP server queried by the `ntp` collector. |

The default only works if the managed host runs an NTP server that answers on localhost, for example chronyd with a matching `allow` directive. Otherwise the `ntp` collector fails (`node_scrape_collector_success{collector="ntp"} 0`) while all other metrics continue to work. Set the variable to an NTP server reachable in your environment, for example in `group_vars`.

All other exporter settings are in `roles/node_exporter/templates/etc_sysconfig_node_exporter.j2`.

## Known limitations

- Collectors that need root privileges (for example `slabinfo` and RAPL energy counters) may fail under the unprivileged `node_exporter` user. They report `node_scrape_collector_success 0` and do not stop the exporter.
- Automatic restart after a process failure (`Restart=on-failure`) has not been tested directly.

## Copyright

Copyright (c) 2026 Nikita Grabar. All rights reserved. See [`COPYRIGHT`](COPYRIGHT). No license is granted.
