# PHASE-4 — PostgreSQL + Patroni

Phase phức tạp nhất. **Chia 4a (skeleton) và 4b (logic)** — không gộp một prompt.

## Mục tiêu

Cài PostgreSQL 16 + Patroni 4 lên 3 PG node. Bootstrap thành cụm 1 primary + 2 replica streaming. Tạo user `dify` và database `dify`.

## In scope

- Role `postgresql`: cài binary PG 16 từ PGDG repo, tạo data dir trống (KHÔNG initdb — Patroni làm).
- Role `patroni`: cài Patroni trong venv `/opt/patroni/venv`, sinh `patroni.yml` từ template, systemd unit, start theo thứ tự đúng.
- Orchestration: leader bootstrap trước (Patroni tự initdb), 2 replica join sau (tự basebackup).
- Tạo user `dify` + database `dify` qua `community.postgresql.postgresql_user/postgresql_db` chạy trên primary động.

## Out of scope

- pgBackRest / WAL archiving — day-2.
- PG extension (pg_stat_statements…) — chỉ enable thông qua `patronictl edit-config` sau, không nhồi vào bootstrap.
- TLS giữa client và PG — đã có lớp PgBouncer trong K8s, có thể bổ sung sau.

## Deliverables (4a — skeleton)

```
roles/postgresql/
├── tasks/
│   ├── main.yml
│   ├── repo.yml             # add PGDG repo
│   ├── install.yml          # cài postgresql-16 (KHÔNG enable systemd của nó — Patroni quản)
│   └── prepare.yml          # tạo data dir trống, ownership
├── defaults/main.yml
├── handlers/main.yml
├── meta/main.yml
└── README.md

roles/patroni/
├── tasks/
│   ├── main.yml
│   ├── install.yml          # tạo venv, pip install patroni[etcd3]
│   ├── configure.yml        # template patroni.yml + systemd unit
│   ├── bootstrap_leader.yml # start patroni trên node đầu, đợi /primary 200
│   └── join_replica.yml     # start patroni trên replica, đợi /replica 200
├── templates/
│   ├── patroni.yml.j2
│   └── patroni.service.j2
├── defaults/main.yml
├── handlers/main.yml
├── meta/main.yml
└── README.md
```

## Deliverables (4b — logic)

- Điền nội dung đầy đủ vào templates theo `docs/skills/patroni-spec.md`.
- Orchestration trong `playbooks/database.yml`:
  ```yaml
  - hosts: db_nodes                        # cài binary, prepare — song song
    tasks:
      - import_role: { name: postgresql }
      - import_role: { name: patroni, tasks_from: install }
      - import_role: { name: patroni, tasks_from: configure }

  - hosts: "{{ groups['db_nodes'][0] }}"   # leader trước
    tasks:
      - import_role: { name: patroni, tasks_from: bootstrap_leader }

  - hosts: "{{ groups['db_nodes'][1:] | join(',') }}"
    serial: 1                              # replica từng cái một
    tasks:
      - import_role: { name: patroni, tasks_from: join_replica }

  - hosts: "{{ groups['db_nodes'][0] }}"   # tạo user/db trên primary
    tasks:
      - name: Create application user
        community.postgresql.postgresql_user:
          name: dify
          password: "{{ dify_db_password }}"
          role_attr_flags: LOGIN
          login_user: postgres
          login_unix_socket: /var/run/postgresql
        become_user: postgres
      - name: Create application database
        community.postgresql.postgresql_db:
          name: dify
          owner: dify
          encoding: UTF8
          template: template0
          login_user: postgres
          login_unix_socket: /var/run/postgresql
        become_user: postgres
  ```
- Lưu ý "động xác định primary": sau bootstrap, host[0] CHẮC CHẮN là primary. Nhưng day-2 playbook không thể giả định — phải query `patronictl list -f json` hoặc REST `/cluster` để tìm primary. Đặt task ấy thành `tasks/find_primary.yml` để reuse.

## Context files cần load

**Prompt 4a (skeleton):**
- `docs/CLAUDE.md`
- `docs/ARCHITECTURE.md`
- `docs/skills/project-conventions.md`
- `docs/skills/patroni-spec.md`
- `docs/phases/PHASE-4-patroni.md` (file này — section 4a)

**Prompt 4b (logic):**
- Tất cả như 4a, không cần thêm.

## Implementation notes

- **Proxy**: Hai loại task cần `environment: "{{ proxy_env }}"`:
  1. **apt** thêm PGDG repo — apt proxy đã do `os_prep` cấu hình, không cần thêm.
  2. **pip** install Patroni venv — `ansible.builtin.pip` không thừa kế system proxy, phải pass trực tiếp:
  ```yaml
  - ansible.builtin.pip:
      name:
        - "patroni[etcd3]==4.1.3"
        - "psycopg[binary]"
      virtualenv: /opt/patroni/venv
      virtualenv_command: python3 -m venv
    environment: "{{ proxy_env }}"
    tags: [patroni, patroni-install]
  ```

- KHÔNG enable systemd của `postgresql.service` từ PGDG package. Mask nó:
  ```yaml
  - ansible.builtin.systemd:
      name: postgresql
      enabled: no
      state: stopped
      masked: yes
  ```
- Patroni venv:
  ```yaml
  - ansible.builtin.pip:
      name:
        - "patroni[etcd3]==4.1.3"
        - "psycopg[binary]"
      virtualenv: /opt/patroni/venv
      virtualenv_command: python3 -m venv
  ```
- Template `patroni.yml` cần biến: `patroni_name`, `ansible_host`, `groups['etcd']`, `network_cidr`, `k8s_pod_cidr`, các password từ vault.
- Watchdog: nếu test env không có softdog, set `watchdog.mode: off` qua biến `patroni_watchdog_mode` (default `required`). Documenta trong role README.
- Handler restart Patroni phải `serial: 1` ở playbook gọi để không restart đồng thời.
- Sau bootstrap, `pg_hba.conf` được Patroni quản qua DCS. Mọi thay đổi tiếp theo phải qua `patronictl edit-config`.

## Acceptance criteria

1. `ansible-playbook playbooks/database.yml` lần đầu thành công.
2. SSH vào pg1: `/opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list` cho:
   ```
   + Cluster: postgres-ha -+--------+---------+----+-----------+
   | Member | Host     | Role   | State   | TL | Lag in MB |
   +--------+----------+--------+---------+----+-----------+
   | pg1    | 192.168.56.111 | Leader | running |  1 |           |
   | pg2    | 192.168.56.112 | Replica| running |  1 |       0   |
   | pg3    | 192.168.56.113 | Replica| running |  1 |       0   |
   +--------+----------+--------+---------+----+-----------+
   ```
3. Từ Ansible controller: `PGPASSWORD=... psql -h <primary> -U dify -d dify -c 'SELECT 1'` ra 1.
4. `patronictl switchover --leader pg1 --candidate pg2` thành công. List sau đó cho thấy pg2 là Leader.
5. Switch lại về pg1, verify state stable.
6. Kill `-9` PG process trên primary, sau ~30-40s thấy node khác lên làm primary. Restart node cũ → tự rejoin làm replica (qua pg_rewind hoặc basebackup).
7. Apply lại playbook → KHÔNG changed.
8. `ansible-lint` clean.

## Cảnh báo

- Nếu replica `Lag in MB` không tiến về 0 trong 5 phút, kiểm: `wal_keep_size`, network giữa node, disk I/O.
- Nếu `pg_rewind` fail khi rejoin (do `wal_log_hints` không bật), phải `patronictl reinit <node>` để basebackup lại.
