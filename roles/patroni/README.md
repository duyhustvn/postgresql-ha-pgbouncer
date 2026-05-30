# Role: patroni

Deploy Patroni for PostgreSQL HA management with etcd as DCS.

## Purpose

Installs Patroni (via pip into a venv), writes `patroni.yml`, registers the
systemd service, and on the designated bootstrap node triggers `patronictl`
cluster init. All PostgreSQL parameter changes go through
`bootstrap.dcs.postgresql.parameters` — never through direct file edits.

## Variables

### defaults/main.yml (overridable)

| Variable | Default | Description |
|---|---|---|
| `patroni_config_dir` | `/etc/patroni` | Config directory |
| `patroni_log_dir` | `/var/log/patroni` | Log directory |
| `patroni_rest_port` | `8008` | Patroni REST API port (used by HAProxy health check) |
| `patroni_watchdog_mode` | `off` | `required` in production, `off` for testing |

### group_vars/db_nodes/main.yml

| Variable | Description |
|---|---|
| `pg_max_connections` | PostgreSQL `max_connections` (managed by Patroni) |
| `pg_work_mem` | Per-sort work memory |
| `pg_shared_buffers` | Shared buffers |

### Required from group_vars/all

| Variable | Description |
|---|---|
| `patroni_version` | Patroni pip version (e.g. `4.1.3`) |
| `cluster_name` | Patroni cluster name |
| `patroni_ttl` | Leader TTL in seconds |
| `patroni_loop_wait` | Loop wait in seconds |
| `postgres_superuser_password` | PostgreSQL superuser password |
| `postgres_replication_password` | Replication user password |
| `postgres_rewind_password` | pg_rewind user password |

## Dependencies

- `postgresql`
- `etcd`

## Example

```yaml
- name: Deploy Patroni
  hosts: db_nodes
  become: true
  roles:
    - role: patroni
```

## Tags

| Tag | Scope |
|---|---|
| `patroni` | All tasks |
| `patroni-install` | pip venv + binary |
| `patroni-config` | patroni.yml template + systemd unit |
