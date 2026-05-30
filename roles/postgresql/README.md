# Role: postgresql

Installs PostgreSQL 16 from the official PGDG apt repository. Does NOT initdb —
Patroni owns cluster initialisation.

## Variables

### Role defaults (`defaults/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `postgresql_user` | `postgres` | OS user running PostgreSQL |
| `postgresql_group` | `postgres` | OS group |
| `postgresql_data_dir` | `/u01/postgresql` | Base data path (full PGDATA = base/version/main) |
| `postgresql_pgdata` | `{{ postgresql_data_dir }}/{{ postgresql_version }}/main` | Full PGDATA path |
| `postgresql_bin_dir` | `/usr/lib/postgresql/{{ postgresql_version }}/bin` | Binary directory |

### Required from `group_vars/all/main.yml`

| Variable | Description |
|---|---|
| `postgresql_version` | Major version to install (e.g. `"16"`) |
| `proxy_env` | Proxy dict — used by `get_url` when downloading the PGDG signing key |

## Task files

| File | Purpose |
|---|---|
| `repo.yml` | Download PGDG signing key, add apt repository |
| `install.yml` | Install `postgresql-{{ postgresql_version }}` packages; mask PGDG systemd unit |
| `prepare.yml` | Create PGDATA directory (`mode: 0700`, owner `postgres`) |

## Design notes

- The PGDG-installed `postgresql.service` is **masked** immediately after install so it
  can never auto-start or race with Patroni.
- This role creates an empty PGDATA directory. Patroni runs `initdb` on the designated
  leader during bootstrap; replicas receive a basebackup automatically.
- All PostgreSQL configuration (postgresql.conf, pg_hba) is managed by Patroni —
  never edit these files directly.

## Tags

| Tag | Scope |
|---|---|
| `postgresql` | All tasks |
| `postgresql-install` | PGDG repo + package install + service mask |
| `postgresql-config` | PGDATA directory creation |

## Dependencies

- `os_prep` (declared in `meta/main.yml`)
