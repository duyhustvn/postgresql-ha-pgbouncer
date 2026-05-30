# skills/patroni-spec.md

Spec cho roles `postgresql` + `patroni`. Phase liên quan: 4, 6.

## Phiên bản & cài đặt

- PostgreSQL 16 (Ubuntu PGDG repo).
- Patroni 4.0.x cài qua `pip install 'patroni[etcd3]'` trong Python venv `/opt/patroni/venv`.
- Lý do dùng venv: tránh xung đột Python system, dễ nâng cấp Patroni.

## Phân chia role

- **`roles/postgresql`**: chỉ cài binary + cấu hình baseline (`apt` PGDG, không initdb, vì Patroni làm việc đó). Tạo data dir trống `/var/lib/postgresql/16/main`.
- **`roles/patroni`**: cài Patroni vào venv, sinh `patroni.yml`, systemd unit, chạy. Patroni sẽ tự initdb lần đầu trên primary, basebackup trên replica.

## Tham số patroni.yml (bootstrap)

Template `templates/patroni.yml.j2`. Tham số sống động lấy từ inventory:

```yaml
scope: {{ cluster_name }}                    # ví dụ 'dify-pg'
namespace: /service/
name: {{ patroni_name }}                     # pg1/pg2/pg3 từ hostvars

restapi:
  listen: 0.0.0.0:8008
  connect_address: {{ ansible_host }}:8008

etcd3:
  hosts: "{% for h in groups['etcd'] %}{{ hostvars[h].ansible_host }}:2379{% if not loop.last %},{% endif %}{% endfor %}"

bootstrap:
  dcs:
    ttl: {{ patroni_ttl | default(30) }}
    loop_wait: {{ patroni_loop_wait | default(10) }}
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 200
        shared_buffers: 16GB
        effective_cache_size: 48GB
        work_mem: 32MB
        maintenance_work_mem: 2GB
        wal_level: replica
        max_wal_size: 8GB
        min_wal_size: 2GB
        checkpoint_completion_target: 0.9
        checkpoint_timeout: 15min
        wal_log_hints: 'on'
        hot_standby: 'on'
        wal_keep_size: 1GB
        max_wal_senders: 10
        max_replication_slots: 10
        random_page_cost: 1.1
        effective_io_concurrency: 200
        idle_in_transaction_session_timeout: 60s
        huge_pages: try
        log_min_duration_statement: 1000
        log_temp_files: 0
        log_lock_waits: 'on'

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator {{ network_cidr }} scram-sha-256
    - host all             all        {{ network_cidr }} scram-sha-256
    - host all             dify       {{ k8s_pod_cidr }} scram-sha-256

  users:
    admin:
      password: {{ postgres_superuser_password }}
      options: [createrole, createdb]

postgresql:
  listen: 0.0.0.0:5432
  connect_address: {{ ansible_host }}:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication: { username: replicator, password: "{{ postgres_replication_password }}" }
    superuser:   { username: postgres,   password: "{{ postgres_superuser_password }}" }
    rewind:      { username: rewind_user, password: "{{ postgres_rewind_password }}" }
  parameters:
    unix_socket_directories: '/var/run/postgresql'

watchdog:
  mode: required                             # đổi 'off' nếu env test không có softdog
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

> `bootstrap.dcs.*` chỉ được Patroni đọc khi khởi tạo cluster lần đầu (không có key trong etcd). Sau đó muốn thay phải dùng `patronictl edit-config`.

## Bootstrap orchestration

Bootstrap cluster yêu cầu **một node lên trước, hai node sau join**. Pattern Ansible:

```yaml
# playbooks/database.yml (pseudocode)
- hosts: db_nodes
  serial: 1
  tasks:
    - import_role: { name: postgresql }
    - import_role: { name: patroni, tasks_from: install }

- hosts: db_nodes[0]
  tasks:
    - import_role: { name: patroni, tasks_from: bootstrap_leader }
    - name: Wait for leader ready
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}:8008/primary"
        status_code: 200
      retries: 30
      delay: 5

- hosts: db_nodes[1:]
  serial: 1
  tasks:
    - import_role: { name: patroni, tasks_from: join_replica }
    - name: Wait for replica streaming
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}:8008/replica"
        status_code: 200
      retries: 30
      delay: 5
```

`tasks_from: bootstrap_leader` và `tasks_from: join_replica` chia ra trong role để playbook orchestration chọn đúng giai đoạn.

## Systemd unit (patroni.service)

```
[Unit]
Description=Patroni HA
After=syslog.target network.target etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni/patroni.yml
KillMode=process
TimeoutSec=30
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## REST API endpoints (Patroni)

| Endpoint | Mục đích |
|---|---|
| GET `/primary` (200) | Node là primary |
| GET `/replica` (200) | Node là replica healthy |
| GET `/leader` | Alias của primary |
| GET `/health` | Liveness |
| POST `/switchover` | Failover có kế hoạch (cần auth) |
| GET `/cluster` | Trạng thái toàn cluster (JSON) |

HAProxy dùng `/primary`. `patronictl` dùng REST này.

## Watchdog

- `watchdog.mode: required` ép kernel watchdog ping; nếu thiếu device → Patroni từ chối start.
- `modprobe softdog` trong role `os_prep`, persist trong `/etc/modules-load.d/softdog.conf`.

## Users cần tạo (Patroni làm trong bootstrap)

| User | Mục đích | Password biến |
|---|---|---|
| `postgres` | superuser | `vault_postgres_superuser_password` |
| `replicator` | streaming replication | `vault_postgres_replication_password` |
| `rewind_user` | pg_rewind | `vault_postgres_rewind_password` |
| `dify` | application user (Dify connect) | `vault_dify_db_password` |

User `dify` không nằm trong `bootstrap.users` (chỉ chứa admin). Tạo bằng task riêng sau bootstrap, qua `community.postgresql.postgresql_user` trên primary.

## Database `dify`

Tạo sau khi cluster up:

```yaml
- name: Create dify database
  community.postgresql.postgresql_db:
    name: dify
    owner: dify
    encoding: UTF8
    lc_collate: en_US.UTF-8
    lc_ctype: en_US.UTF-8
    template: template0
    state: present
  become_user: postgres
  run_once: true
  delegate_to: "{{ groups['db_nodes'] | first }}"
  when: hostvars[groups['db_nodes'] | first]['patroni_role'] | default('') == 'primary'
```

(Hoặc đơn giản hơn: dùng `patronictl list -f json` để xác định primary động.)

## Acceptance criteria phase 4

- `patronictl -c /etc/patroni/patroni.yml list` cho ra 1 Leader + 2 Replica, state `running`, `Lag in MB: 0`.
- `psql -h <primary> -U dify -d dify -c 'SELECT 1'` từ Ansible controller (qua jump nếu cần) trả về 1.
- Manual `patronictl switchover` thành công, leader đổi sang node khác, sau đó switch lại.
- Apply lại playbook không có changed.
