# Project Context

Durable handoff state for agents. See `AGENTS.md` for what belongs here.

## Current state

- The role installs node_exporter from the RED OS package `prometheus-node_exporter` and manages its packaged systemd units. It replaced an earlier Docker-based deployment.
- Manually tested on RED OS 7.3 and RED OS 8, with and without the package already installed: deployment, `--check`, working `/metrics`, and `changed=0` on the second run.
- Syntax check and yamllint pass. ansible-lint reports no violations, though some of its rules raised internal exceptions.
- Not directly tested: automatic restart (`Restart=on-failure`) after a node_exporter process failure.

## Decisions and constraints

- Package facts (verified on both OSes): units `node_exporter.service` and `node_exporter.socket` (`ListenStream=9100`), `ExecStart=/usr/sbin/node_exporter --web.systemd-socket $OPTIONS`, `EnvironmentFile` `/etc/sysconfig/node_exporter`, user `node_exporter`; node_exporter 1.10.2 on 7.3 and 1.12.1 on 8. No OS-specific branches.
- No backslashes, quotes or whitespace inside `OPTIONS` argument values. systemd 247 and later keep `\.` in double-quoted env values but 239 drops the backslash, and `$OPTIONS` splitting (EXTRACT_UNQUOTE) strips backslashes on all versions. Use `[.]` instead of `\.`. Verified from systemd sources.
- Never add `--web.listen-address`; the port belongs to the packaged socket unit.
- Check mode: unit enable/start and the restart handler are skipped when the package install is only simulated (the units don't exist yet); the `/metrics` check is always skipped (uri returns no status in check mode).
- `node_exporter_ntp_server` defaults to `127.0.0.1`, which needs a local NTP server; deployments override it.
- Known, accepted collector failures: collectors needing root (e.g. `slabinfo`, RAPL energy) may fail under the unprivileged `node_exporter` user. They show as `node_scrape_collector_success=0` and don't stop the exporter.

## Next steps / blockers

_None recorded._
