# Role: patroni

Installs Patroni into a Python venv, templates `patroni.yml` and the systemd unit,
and provides entry-point task files for leader bootstrap and replica join.

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                   Patroni HA Cluster (3 nodes)                     │
│                                                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │       pg1        │  │       pg2        │  │       pg3        │  │
│  │  192.168.56.111  │  │  192.168.56.112  │  │  192.168.56.113  │  │
│  │                  │  │                  │  │                  │  │
│  │ ┌──────────────┐ │  │ ┌──────────────┐ │  │ ┌──────────────┐ │  │
│  │ │   Patroni    │ │  │ │   Patroni    │ │  │ │   Patroni    │ │  │
│  │ │  REST :8008  │ │  │ │  REST :8008  │ │  │ │  REST :8008  │ │  │
│  │ └──────┬───────┘ │  │ └──────┬───────┘ │  │ └──────┬───────┘ │  │
│  │        │         │  │        │         │  │        │         │  │
│  │ ┌──────▼───────┐ │  │ ┌──────▼───────┐ │  │ ┌──────▼───────┐ │  │
│  │ │  PostgreSQL  │ │  │ │  PostgreSQL  │ │  │ │  PostgreSQL  │ │  │
│  │ │   :5432      │ │  │ │   :5432      │ │  │ │   :5432      │ │  │
│  │ │  [PRIMARY]   │ │  │ │  [REPLICA]   │ │  │ │  [REPLICA]   │ │  │
│  │ └──────────────┘ │  │ └──────▲───────┘ │  │ └──────▲───────┘ │  │
│  └──────────────────┘  └────────┼─────────┘  └────────┼─────────┘  │
│           │                     │ streaming             │ streaming  │
│           └─────────────────────┴───────────────────────┘           │
│                         WAL replication                              │
└────────────────────────────────────────────────────────────────────┘
                   │                   │                   │
           ┌───────▼───────────────────▼───────────────────▼───────┐
           │              etcd Cluster (DCS)                        │
           │   etcd1:2379    etcd2:2379    etcd3:2379               │
           │          Leader election lock / cluster config          │
           └────────────────────────────────────────────────────────┘
```

**Cơ chế hoạt động:**
- Patroni sử dụng etcd làm DCS để giữ distributed lock cho leader.
- Chỉ node nắm lock mới được chạy PostgreSQL ở vai trò PRIMARY.
- Khi primary bị mất kết nối với etcd quá TTL (`patroni_ttl: 30s`), lock được giải phóng
  và một replica sẽ thực hiện leader election và promote lên PRIMARY.
- HAProxy health-check gọi `GET :8008/primary` (HTTP 200) và `GET :8008/replica` (HTTP 200)
  để định tuyến kết nối đúng node.

## Kiểm tra trạng thái (Status Commands)

### Xem tổng quan cluster (khuyến nghị)

```bash
# Hiển thị tất cả member: tên, host, role, state, replication lag
patronictl -c /etc/patroni/patroni.yml list
```

Kết quả mong đợi:
```
+ Cluster: postgres-ha (xxxxxxxxxxxxxxxx) +---------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+--------+-----------------+---------+---------+----+-----------+
| pg1    | 192.168.56.111  | Leader  | running |  1 |           |
| pg2    | 192.168.56.112  | Replica | streaming |  1 |         0 |
| pg3    | 192.168.56.113  | Replica | streaming |  1 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

### Xem topology dạng cây

```bash
patronictl -c /etc/patroni/patroni.yml topology
```

### Kiểm tra REST API từng node

```bash
# Node hiện tại có phải primary không? (200 = yes, 503 = no)
curl -s http://192.168.56.111:8008/primary
curl -s http://192.168.56.112:8008/primary
curl -s http://192.168.56.113:8008/primary

# Node hiện tại có phải replica không? (200 = yes, 503 = no)
curl -s http://192.168.56.111:8008/replica
curl -s http://192.168.56.112:8008/replica
curl -s http://192.168.56.113:8008/replica

# Thông tin sức khoẻ tổng hợp của một node
curl -s http://192.168.56.111:8008/health | python3 -m json.tool

# Topology toàn cluster qua một node bất kỳ
curl -s http://192.168.56.111:8008/cluster | python3 -m json.tool
```

### Kiểm tra systemd service

```bash
# Trạng thái service (chạy trực tiếp trên từng host)
systemctl status patroni

# Xem log Patroni (leader election, failover, config reload)
journalctl -u patroni -n 100 --no-pager

# Theo dõi log real-time
journalctl -u patroni -f
```

### Kiểm tra nhanh qua Ansible

```bash
# Chạy lại bước verify mà không deploy lại
ansible-playbook playbooks/patroni.yml --tags patroni-verify
```

### Thao tác quản trị thường dùng (patronictl)

```bash
# Xem cấu hình cluster hiện tại (TTL, loop_wait, ...)
patronictl -c /etc/patroni/patroni.yml show-config

# Xem lịch sử failover / switchover
patronictl -c /etc/patroni/patroni.yml history

# Thực hiện planned switchover (chuyển primary sang node khác)
patronictl -c /etc/patroni/patroni.yml switchover --master pg1 --candidate pg2

# Restart Patroni trên một node (an toàn hơn systemctl restart)
patronictl -c /etc/patroni/patroni.yml restart postgres-ha pg1
```

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
