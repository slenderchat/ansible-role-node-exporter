# ansible-role-node-exporter

[English](README.md) | [Русский](README.ru.md)

Ansible role for installing and configuring [Prometheus Node Exporter](https://github.com/prometheus/node_exporter) on RED OS 7.3 and RED OS 8 using distribution-provided RPM packages and systemd service/socket units.

Implemented by Nikita Grabar.

> **Copyright.** Copyright (c) 2026 Nikita Grabar. All rights reserved. This repository is not distributed under an open-source license. Using, copying, modifying or redistributing it requires the prior permission of the copyright holder. See [COPYRIGHT](COPYRIGHT).

## Supported platforms

- RED OS 7.3 (Node Exporter 1.10.2 tested).
- RED OS 8 (Node Exporter 1.12.1 tested).

Deployment was validated on both operating systems, including hosts with and without the package already installed. The initial deployment, repeated run (`changed=0`), check mode and `/metrics` availability were verified.

## Requirements

- Ansible 2.10 or later on the control node (developed with ansible-core 2.19).
- The RPM package `prometheus-node_exporter` must be available from the managed host's configured repositories.
- Systemd with the package-provided `node_exporter.service` and `node_exporter.socket` units.
- TCP port `9100` must not be occupied by another exporter or container.
- Privilege escalation (`become: true`) to install the package and manage service configuration.

## Installation

Install this standalone repository under the role name `node_exporter` in an Ansible roles path:

```bash
mkdir -p roles
git clone https://github.com/slenderchat/ansible-role-node-exporter.git roles/node_exporter
```

You can also configure `roles_path` or `ANSIBLE_ROLES_PATH` to point to the parent directory. There are no role dependencies.

## Configuration

| Variable | Default | Description |
|---|---|---|
| `node_exporter_ntp_server` | `127.0.0.1` | NTP server queried by the NTP collector. |

The default works only if the managed host runs an NTP server that answers on localhost, for example chronyd configured to accept the query. Otherwise `node_scrape_collector_success{collector="ntp"} 0` is reported while other collectors continue to work. Set the variable to an accessible NTP server in the host or group variables where appropriate.

Other exporter arguments are defined in `templates/etc_sysconfig_node_exporter.j2`. The template enables host and systemd collectors including textfile, NTP, processes, filesystem, networking, memory and CPU metrics. Collector filters and detailed options are maintained in that template. The package's textfile collector directory is `/var/lib/node_exporter/textfile_collector`.

### Argument escaping

The template stores arguments in `OPTIONS` in `/etc/sysconfig/node_exporter`. Systemd parses them first as an `EnvironmentFile` value and then splits `$OPTIONS` into arguments in `ExecStart`. As a result, argument values must not contain whitespace, double or single quotes, or backslashes. Use `[.]` instead of `\.` in regular expressions; see the existing template for examples.

## Usage

Create a deployment playbook outside this role repository, for example `site.yml`:

```yaml
---
- name: Deploy Node Exporter
  hosts: all
  become: true
  gather_facts: false
  roles:
    - node_exporter
```

Preview changes first, then deploy to the intended hosts:

```bash
ANSIBLE_ROLES_PATH=./roles ansible-playbook -i inventory.ini site.yml --limit HOST --check --diff
ANSIBLE_ROLES_PATH=./roles ansible-playbook -i inventory.ini site.yml --limit HOST --diff
```

The supplied playbook uses `hosts: all`, so the `--limit` option is recommended for staged deployments.

### What the role does

1. Installs the package `prometheus-node_exporter` using `state: present`. The role does not pin a version or automatically upgrade an installed package.
2. Unmasks the packaged `node_exporter.socket` and `node_exporter.service`, enables them at boot and starts them.
3. Writes collector arguments into `/etc/sysconfig/node_exporter` (`OPTIONS`).
4. Installs `/etc/systemd/system/node_exporter.service.d/restart.conf` with `Restart=on-failure`, without changing packaged systemd units.
5. Reloads systemd when the drop-in changes and restarts the service when options or the drop-in change. Unchanged configuration does not trigger a restart.
6. Checks for HTTP 200 at `http://127.0.0.1:9100/metrics` on normal deployments.

The listen port is defined by the packaged `node_exporter.socket` (`ListenStream=9100`). The exporter uses socket activation (`--web.systemd-socket`); **do not add** `--web.listen-address` to `OPTIONS`. The service runs as the unprivileged `node_exporter` system user from the package.

## Validation

After deployment, verify the service/socket state and metrics endpoint:

```bash
systemctl is-enabled node_exporter.socket node_exporter.service
systemctl is-active node_exporter.socket node_exporter.service
systemctl status node_exporter.service --no-pager
curl --noproxy '*' -fsS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:9100/metrics
```

A second Ansible run should report `changed=0`. Check mode (`--check --diff`) supports hosts without the RPM installed: unit enable/start and restart actions are skipped when package installation is only simulated. The HTTP check is always skipped in check mode.

When migrating from Docker, stop the old container and free port `9100` before starting the packaged socket; Docker lifecycle and rollback are outside this role.

## Known limitations

- Some collectors, including `slabinfo` and RAPL energy metrics, may lack permissions under the unprivileged service user. They report `node_scrape_collector_success 0` without stopping the exporter.
- The NTP collector needs a reachable NTP server. Its failure does not prevent other metrics from being exported.
- Automatic recovery through `Restart=on-failure` is configured but has not been directly verified by deliberately killing the exporter process.
- A successful HTTP response does not establish that every collector succeeded; inspect `node_scrape_collector_success` for individual collectors.

## Copyright

Copyright (c) 2026 Nikita Grabar. All rights reserved. No license is granted; see [COPYRIGHT](COPYRIGHT).
