# Role: postgresql

Install PostgreSQL 16 from the official PGDG apt repository.

## Purpose

Installs the `postgresql-16` package, creates the OS user, and sets up the data
directory under `/u01/postgresql`. Does **not** initialize the cluster — that is
Patroni's responsibility via `initdb` on bootstrap.

> **Important**: This role does NOT touch `postgresql.conf` or `pg_hba.conf`
> directly. All PostgreSQL configuration goes through Patroni (bootstrap.dcs).

## Variables

### defaults/main.yml (overridable)

| Variable | Default | Description |
|---|---|---|
| `postgresql_user` | `postgres` | OS user running PostgreSQL |
| `postgresql_group` | `postgres` | OS group |
| `postgresql_data_dir` | `/u01/postgresql` | PGDATA directory |
| `postgresql_log_dir` | `/u01/postgresql/log` | Log directory |
| `postgresql_port` | `5432` | Listening port |

### Required from group_vars/all

| Variable | Description |
|---|---|
| `postgresql_version` | Major version to install (e.g. `"16"`) |

## Dependencies

- `os_prep`

## Example

```yaml
- name: Install PostgreSQL 16
  hosts: db_nodes
  become: true
  roles:
    - role: postgresql
```

## Tags

| Tag | Scope |
|---|---|
| `postgresql` | All tasks |
| `postgresql-install` | Package and directory setup |
| `postgresql-config` | pg_hba stub (Patroni overrides on bootstrap) |
