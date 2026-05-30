# Role: patroni

Installs Patroni into a Python venv, templates `patroni.yml` and the systemd unit,
and provides entry-point task files for leader bootstrap and replica join.

## Variables

### Role defaults (`defaults/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `patroni_venv` | `/opt/patroni/venv` | Python virtual environment path |
| `patroni_config_dir` | `/etc/patroni` | Config directory |
| `patroni_log_dir` | `/var/log/patroni` | Log directory |
| `patroni_watchdog_mode` | `required` | `required` in production, `off` for testing (see note) |
| `network_cidr` | `192.168.56.0/24` | VM subnet — used in pg_hba; override in group_vars |

### Required from `group_vars/all/main.yml`

| Variable | Description |
|---|---|
| `patroni_version` | Patroni pip version (e.g. `4.1.3`) |
| `cluster_name` | Patroni cluster scope |
| `patroni_ttl` | Leader TTL in seconds |
| `patroni_loop_wait` | Loop wait in seconds |
| `patroni_rest_port` | REST API port (HAProxy health-check target) |
| `etcd_client_port` | etcd client port used in `etcd3.hosts` |
| `k8s_pod_cidr` | Kubernetes pod CIDR for pg_hba dify rule |
| `proxy_env` | Proxy dict — used by `pip install` |
| `postgres_superuser_password` | Mapped from vault |
| `postgres_replication_password` | Mapped from vault |
| `postgres_rewind_password` | Mapped from vault |

### From `postgresql` role defaults (loaded via meta dependency)

| Variable | Description |
|---|---|
| `postgresql_user / group` | Unix user/group for ownership |
| `postgresql_pgdata` | Full PGDATA path passed to `patroni.yml` |
| `postgresql_bin_dir` | PostgreSQL binaries path |
| `postgresql_port` | PostgreSQL listen port |

## Task files (entry points)

| File | Called via | Purpose |
|---|---|---|
| `install.yml` | `main.yml` or `tasks_from: install` | venv creation + pip install |
| `configure.yml` | `main.yml` or `tasks_from: configure` | Template patroni.yml + systemd unit |
| `bootstrap_leader.yml` | `tasks_from: bootstrap_leader` | Start Patroni on node[0], wait `/primary` |
| `join_replica.yml` | `tasks_from: join_replica` | Start Patroni on replica, wait `/replica` |

## Watchdog note

`patroni_watchdog_mode: required` is the safe production default — Patroni will refuse to
start if `/dev/watchdog` is not available. For test environments without `softdog`,
override in `group_vars/db_nodes/main.yml`:

```yaml
patroni_watchdog_mode: "off"
```

The `os_prep` role loads `softdog` via `/etc/modules-load.d/softdog.conf` on production nodes.

## Day-2 restart safety

The `restart patroni` handler (daemon-reload → service restart) fires for config changes.
**Never restart all three nodes simultaneously** — it causes a temporary loss of primary.
Always set `serial: 1` on the calling play for rolling restarts.

## Tags

| Tag | Scope |
|---|---|
| `patroni` | All tasks |
| `patroni-install` | venv + pip |
| `patroni-config` | patroni.yml + systemd unit |
| `patroni-start` | bootstrap_leader + join_replica |

## Dependencies

- `postgresql` (declared in `meta/main.yml`)
- `etcd` (declared in `meta/main.yml`)
