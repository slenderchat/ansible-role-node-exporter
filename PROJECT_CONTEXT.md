# Project Context

Durable handoff state for agents. See `AGENTS.md` for what belongs here.

## Current state

- This repository is a standalone `node_exporter` Ansible role with `tasks/`, `handlers/`, `defaults/`, `files/` and `templates/` at the root, not a playbook project. Consumers supply their own playbook and place this repository under the role name `node_exporter` in their roles path.
- The role installs the RED OS `prometheus-node_exporter` package and manages its packaged systemd service and socket units; it replaced an earlier Docker-based deployment.
- Runtime behavior was manually validated on RED OS 7.3 and RED OS 8 with and without the package preinstalled: deployment, check mode, working `/metrics`, and `changed=0` on repeat runs. The repository-layout migration did not change role implementation files; runtime validation after migration has not been performed separately.
- Automatic recovery after a deliberately induced process failure (`Restart=on-failure`) has not been directly tested.

## Decisions and constraints

- Verified package units: `node_exporter.service` and `node_exporter.socket` (`ListenStream=9100`); `ExecStart=/usr/sbin/node_exporter --web.systemd-socket $OPTIONS`, `EnvironmentFile` `/etc/sysconfig/node_exporter`, user `node_exporter`. Tested exporter versions: 1.10.2 on RED OS 7.3 and 1.12.1 on RED OS 8. No OS-specific branches.
- Do not add `--web.listen-address`; the port belongs to the packaged socket unit.
- Argument values inside `OPTIONS` cannot contain backslashes, quotes or whitespace: systemd 247+ may preserve `\.` inside double-quoted environment values while older versions may not, and `$OPTIONS` argument splitting drops backslashes. Use `[.]` instead of `\.`. This was verified against systemd behavior and sources.
- Check mode: enable/start operations and the restart handler are skipped if package installation was only simulated (units do not yet exist); the `/metrics` check always skips in check mode.
- `node_exporter_ntp_server` defaults to `127.0.0.1` and normally needs an accessible local NTP server; deployments may override it.
- Accepted collector limitations: `slabinfo`, RAPL and other collectors needing extra permissions can report `node_scrape_collector_success=0` under the unprivileged `node_exporter` user without preventing service operation.
- All rights reserved (`COPYRIGHT`); do not introduce an open-source license or Ansible Galaxy license metadata without the copyright holder's approval.

## Validation

Run from the repository root. Build a temporary playbook because no playbook is shipped:

```bash
tmp=$(mktemp -d)
mkdir -p "$tmp/roles"
ln -s "$PWD" "$tmp/roles/node_exporter"
printf -- '---\n- hosts: all\n  gather_facts: false\n  roles:\n    - node_exporter\n' > "$tmp/site.yml"
ANSIBLE_ROLES_PATH="$tmp/roles" ansible-playbook -i localhost, "$tmp/site.yml" --syntax-check
rm -rf "$tmp"
ansible-lint --profile production .
yamllint -d '{extends: default, rules: {line-length: {max: 120}}}' .
```

Compare runtime files against the last verified release when performing documentation/layout-only changes. Real systemd socket activation and collector behavior must be tested on RED OS hosts.
