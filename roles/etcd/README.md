# Role: etcd

Deploys a 3-node etcd cluster as the DCS (Distributed Configuration Store) for Patroni.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    etcd Cluster (3 nodes)                    │
│                                                              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│  │    etcd1    │   │    etcd2    │   │    etcd3    │        │
│  │ pg1         │◄─►│ pg2         │◄─►│ pg3         │        │
│  │192.168.56.111│   │192.168.56.112│   │192.168.56.113│       │
│  │             │   │             │   │             │        │
│  │ client:2379 │   │ client:2379 │   │ client:2379 │        │
│  │ peer:  2380 │   │ peer:  2380 │   │ peer:  2380 │        │
│  └─────────────┘   └─────────────┘   └─────────────┘        │
│         ▲                  ▲                 ▲               │
│         └──────────────────┴─────────────────┘               │
│                    Raft consensus                             │
└──────────────────────────────────────────────────────────────┘
                             │
                      DCS lock / config
                             │
                    Patroni (all 3 nodes)
```

**Quorum:** requires 2/3 nodes healthy. Losing 2 nodes halts all writes.

## Kiểm tra trạng thái (Status Commands)

Tất cả lệnh `etcdctl` cần biến môi trường `ETCDCTL_API=3`. Đặt endpoints đầy đủ để
kiểm tra toàn cluster thay vì chỉ một node.

```bash
export ETCDCTL_API=3
export ENDPOINTS=http://192.168.56.111:2379,http://192.168.56.112:2379,http://192.168.56.113:2379
```

### Kiểm tra sức khoẻ cluster

```bash
# Kiểm tra từng node có healthy và cùng chung một cluster không
etcdctl --endpoints=$ENDPOINTS endpoint health --cluster
```

Kết quả mong đợi:
```
http://192.168.56.111:2379 is healthy: successfully committed proposal: took = ...
http://192.168.56.112:2379 is healthy: successfully committed proposal: took = ...
http://192.168.56.113:2379 is healthy: successfully committed proposal: took = ...
```

### Xem trạng thái chi tiết từng node

```bash
# Hiển thị leader, raft term, raft index, DB size dưới dạng bảng
etcdctl --endpoints=$ENDPOINTS endpoint status --write-out=table
```

Kết quả mong đợi (cột `IS LEADER` = true chỉ đúng với 1 node):
```
+---------------------------+------------------+---------+---------+-----------+------------+-----------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+
| http://192.168.56.111:2379 | xxxxxxxxxxxxxxxx |  3.5.30 |   20 kB |     false |      false |         5 |
| http://192.168.56.112:2379 | xxxxxxxxxxxxxxxx |  3.5.30 |   20 kB |      true |      false |         5 |
| http://192.168.56.113:2379 | xxxxxxxxxxxxxxxx |  3.5.30 |   20 kB |     false |      false |         5 |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+
```

### Xem danh sách thành viên

```bash
# Danh sách tất cả member: ID, tên, peer URL, client URL
etcdctl --endpoints=$ENDPOINTS member list --write-out=table
```

### Kiểm tra systemd service

```bash
# Trạng thái service trên từng node (chạy trực tiếp trên host)
systemctl status etcd

# Xem log service
journalctl -u etcd -n 50 --no-pager
```

### Kiểm tra nhanh qua Ansible

```bash
# Chạy lại bước verify mà không cần deploy lại
ansible-playbook playbooks/etcd.yml --tags etcd-verify
```

## Variables

### Required (set in `group_vars/all/main.yml`)

| Variable | Example | Description |
|---|---|---|
| `etcd_version` | `v3.5.30` | etcd release tag to download from GitHub |
| `cluster_name` | `postgres-ha` | Used as `initial-cluster-token` prefix |
| `etcd_client_port` | `2379` | Client listen port |
| `etcd_peer_port` | `2380` | Peer communication port |
| `proxy_env` | `{http_proxy: ""}` | Proxy dict for internet-facing tasks |

### Required per host (set in `inventories/prod/hosts.yml`)

| Variable | Example | Description |
|---|---|---|
| `etcd_name` | `etcd1` | Unique name for this etcd member |
| `ansible_host` | `192.168.56.111` | IP used in listen/advertise URLs |

### Role defaults (`defaults/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `etcd_user` | `etcd` | System user running etcd |
| `etcd_group` | `etcd` | System group |
| `etcd_install_dir` | `/usr/local/bin` | Binary destination |
| `etcd_data_dir` | `/u01/etcd` | Data directory (must be on /u01 partition) |
| `etcd_config_dir` | `/etc/etcd` | Config directory |
| `etcd_archive_checksum` | `""` | Set to `sha256:<hash>` to verify download |
| `etcd_health_check_retries` | `12` | Retry count for cluster health check |
| `etcd_health_check_delay` | `5` | Seconds between retries |

## Task files

| File | Purpose |
|---|---|
| `install.yml` | Create user/group, download binary, install to PATH |
| `configure.yml` | Create directories, template config + systemd unit |
| `start.yml` | Bootstrap detection, daemon-reload, enable+start service |
| `verify.yml` | `etcdctl endpoint health` with retry (run_once) |

## Tags

| Tag | Scope |
|---|---|
| `etcd` | All tasks |
| `etcd-install` | Binary download and user/group setup |
| `etcd-config` | Config file and systemd unit templating |
| `etcd-start` | Service enable/start |
| `etcd-verify` | Health check |

## Usage

```bash
# Bootstrap (all nodes parallel):
ansible-playbook playbooks/etcd.yml

# Day-2 config change (rolling restart, one node at a time):
# Add "serial: 1" to the first play in etcd.yml, then:
ansible-playbook playbooks/etcd.yml

# Targeted install only:
ansible-playbook playbooks/etcd.yml --tags etcd-install

# Verify only:
ansible-playbook playbooks/etcd.yml --tags etcd-verify
```

## Day-2 restart safety

The `restart etcd` handler reloads systemd then restarts the service. On a live
cluster, **never restart all three nodes simultaneously** — losing quorum causes
leader election to stall. Always set `serial: 1` on the deploy play when a config
change needs a rolling restart.

## Dependencies

- `os_prep` role (declared in `meta/main.yml`) — verifies `/u01` is mounted and
  applies OS-level prerequisites before etcd is installed.
