# CLAUDE.md — Dify PG HA Ansible Project

## Mục tiêu dự án

Sinh và duy trì một dự án **Ansible** triển khai cụm **PostgreSQL HA** cho ứng dụng **Dify**, gồm 4 tầng kỹ thuật:

1. **etcd** (3 node) — DCS cho Patroni
2. **PostgreSQL 16 + Patroni** (3 node) — primary + 2 replica, tự failover
3. **HAProxy** (K8s Deployment, 2 replica) — route tới primary qua Patroni REST `/primary`
4. **PgBouncer** (K8s Deployment, 3 replica) — transaction-mode connection pooling

Dify (đã chạy trong K8s) sẽ connect vào K8s Service của PgBouncer.

## Quy tắc nền tảng (bắt buộc tuân)

1. **Idempotent**. Mọi task Ansible chạy lại lần 2 phải không đổi gì (no changed). Dùng `creates`/`removes`/`when` thay vì shell raw.
2. **Patroni sở hữu PG config**. KHÔNG sửa `postgresql.conf` trực tiếp, KHÔNG dùng `ALTER SYSTEM`. Mọi thay đổi PG đi qua `patronictl edit-config` hoặc qua `bootstrap.dcs.postgresql.parameters` trong patroni.yml.
3. **Không hardcode IP**. Mọi địa chỉ lấy từ inventory (`hostvars`, `groups`).
4. **Secrets qua ansible-vault**. Mật khẩu, SCRAM verifier nằm trong `group_vars/all/vault.yml` (encrypted). Không bao giờ commit plaintext.
5. **Role-based**. Mỗi tầng = 1 role. Roles dùng được độc lập, không reach vào nhau qua đường dẫn cứng.
6. **Lint sạch**. `ansible-lint` chạy không warning ở mức profile `production`.
7. **Failover-safe Day-2**. Mọi playbook ngoài cài đặt ban đầu (switchover, reinit, vacuum…) phải an toàn khi primary đổi.

## Tech stack

- Ansible >= 2.16 (controller); Python 3.10+ trên target nodes
- PostgreSQL 16, Patroni 4.x, etcd v3.5
- HAProxy 2.9+, PgBouncer 1.21+
- Collections: `kubernetes.core`, `community.postgresql`, `ansible.posix`, `community.general`
- Target OS: Ubuntu 22.04 LTS (giả định mặc định; biến hóa được)

## Bố cục repo (sau khi Phase 1 scaffold xong)

```
.
├── ansible.cfg
├── requirements.yml                # collections
├── inventories/
│   └── prod/
│       ├── hosts.yml
│       └── group_vars/
│           ├── all/
│           │   ├── main.yml
│           │   └── vault.yml       # encrypted
│           ├── db_nodes/
│           │   └── main.yml
│           └── etcd/
│               └── main.yml
├── playbooks/
│   ├── site.yml                    # gọi tất cả role theo thứ tự
│   ├── os_prep.yml
│   ├── etcd.yml
│   ├── database.yml                # postgres + patroni
│   ├── k8s_frontend.yml            # haproxy + pgbouncer
│   ├── verify.yml
│   ├── switchover.yml
│   └── reinit_replica.yml
├── roles/
│   ├── os_prep/
│   ├── etcd/
│   ├── postgresql/
│   ├── patroni/
│   ├── haproxy/
│   └── pgbouncer/
└── docs/                           # bundle này
```

## Nguyên tắc làm việc với Claude Code

- **Mỗi prompt một mục tiêu.** Không chèn nhiều phase vào cùng một prompt.
- **Tải tối thiểu context.** Prompt chỉ load CLAUDE.md + ARCHITECTURE.md + skill cần + phase đang làm.
- **Hỏi khi không chắc.** Khi spec mơ hồ, dừng lại hỏi thay vì đoán — đặc biệt với secret, IP, version pin.
- **Đầu ra phải lint pass.** Kết thúc phase phải chạy được `ansible-lint` không warning.

## Proxy support

Khi deploy trong môi trường corporate có HTTP proxy, dùng **per-task proxy** (không ghi hệ thống toàn cục) để tránh ảnh hưởng traffic nội bộ cluster (etcd peer, Patroni REST, PG replication).

**Biến trung tâm** (khai báo trong `group_vars/all/main.yml`, rỗng khi không dùng proxy):

```yaml
http_proxy: ""
https_proxy: ""
no_proxy: "localhost,127.0.0.1,{{ network_cidr }}"
proxy_env:           # dict dùng chung — không ghi đè trực tiếp
  http_proxy: "{{ http_proxy }}"
  https_proxy: "{{ https_proxy if https_proxy else http_proxy }}"
  no_proxy: "{{ no_proxy }}"
```

**Áp dụng theo tầng:**

| Tầng | Cơ chế | Task áp dụng |
|---|---|---|
| `apt` | `/etc/apt/apt.conf.d/95ansible-proxy` | Tự động qua `os_prep` khi `http_proxy` non-empty |
| `get_url` (etcd binary) | `environment: "{{ proxy_env }}"` | `etcd/tasks/install.yml` |
| `pip` (Patroni venv) | `environment: "{{ proxy_env }}"` | `patroni/tasks/install.yml` |
| K8s image pull | systemd drop-in containerd | Ngoài scope Ansible bundle — cấu hình khi provision K8s |

**Quy tắc:** Mọi task dùng `get_url`, `pip`, hoặc `uri` mà fetch từ internet **phải** có `environment: "{{ proxy_env }}"`. Task nội bộ cluster (etcdctl, patronictl, psql) **không** thêm `environment` này.

## Tham chiếu chi tiết

- Kiến trúc: `ARCHITECTURE.md`
- Phase plan: `PHASES.md`
- Spec từng thành phần: `skills/*.md`
- Prompt sẵn dùng: `PROMPTS.md`
