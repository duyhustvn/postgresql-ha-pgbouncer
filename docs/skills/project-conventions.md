# skills/project-conventions.md

Ràng buộc style + tổ chức repo. Load skill này ở MỌI phase.

## Layout role (chuẩn Ansible)

```
roles/<name>/
├── defaults/main.yml        # giá trị mặc định, có thể override
├── vars/main.yml            # hằng số nội bộ role, không nên override
├── tasks/main.yml           # entrypoint; tasks dài chia sub-files include_tasks
├── handlers/main.yml
├── templates/               # *.j2
├── files/                   # static files
├── meta/main.yml            # dependencies, supported platforms
└── README.md                # tham số đầu vào, ví dụ sử dụng
```

## Inventory layout

```
inventories/prod/
├── hosts.yml                # khai báo groups: db_nodes, etcd, k8s_admin
└── group_vars/
    ├── all/
    │   ├── main.yml         # biến chung (versions, ports, network CIDR)
    │   └── vault.yml        # ENCRYPTED — passwords, SCRAM verifiers
    ├── db_nodes/
    │   └── main.yml         # PG/Patroni params
    └── etcd/
        └── main.yml         # etcd cluster params
```

### hosts.yml mẫu cấu trúc (không phải IP thật):

```yaml
all:
  children:
    db_nodes:
      hosts:
        pg1: { ansible_host: 192.168.56.111, patroni_name: pg1, etcd_name: etcd1 }
        pg2: { ansible_host: 192.168.56.112, patroni_name: pg2, etcd_name: etcd2 }
        pg3: { ansible_host: 192.168.56.113, patroni_name: pg3, etcd_name: etcd3 }
    etcd:
      hosts:
        pg1: {}
        pg2: {}
        pg3: {}
    k8s_admin:
      hosts:
        localhost:
          ansible_connection: local
```

### group_vars/all/main.yml chứa (không exhaustive):

```yaml
cluster_name: postgres-ha
postgresql_version: "16"
patroni_version: "4.1.3"
etcd_version: "v3.5.30"
haproxy_image: "haproxy:2.9"
pgbouncer_image: "edoburu/pgbouncer:1.23"
k8s_namespace: default
k8s_pod_cidr: "10.244.0.0/16"
patroni_ttl: 30
patroni_loop_wait: 10
```

### group_vars/all/vault.yml (encrypted) chứa:

```yaml
vault_postgres_superuser_password: "..."
vault_postgres_replication_password: "..."
vault_postgres_rewind_password: "..."
vault_dify_db_password: "..."
vault_pgbouncer_admin_password: "..."
```

Trong playbook tham chiếu qua biến không-prefix-vault:
```yaml
postgres_superuser_password: "{{ vault_postgres_superuser_password }}"
```
(quy ước: vault.yml lưu `vault_<name>`, file main.yml map sang biến không có prefix vault).

## Quy tắc idempotency

1. **Không dùng `shell` hay `command` raw** khi có module thay thế. Nếu phải dùng, kèm `creates:`, `removes:`, hoặc `changed_when:` rõ ràng.
2. **Mọi service dùng `state: started` + `enabled: yes`** (không `state: restarted` trong task chính — restart phải qua handler).
3. **Template file phải có `notify:` handler** restart/reload service khi đổi.
4. **Package install** dùng module phù hợp (`apt`/`dnf`), tránh `pip` global trừ trong venv.
5. **File ownership/permissions** luôn khai báo (`owner`, `group`, `mode`).
6. **Check mode (`--check`) phải chạy được không lỗi.**

## Secrets handling

- Tạo vault: `ansible-vault create inventories/prod/group_vars/all/vault.yml`
- Edit: `ansible-vault edit ...`
- Vault password lưu trong file `.vault_pass` (gitignored), ansible.cfg trỏ tới:
  ```
  [defaults]
  vault_password_file = .vault_pass
  ```
- KHÔNG echo password ra log: dùng `no_log: true` cho task chứa secret trong command line.
- Đối với K8s Secret: render từ template Jinja, apply qua `kubernetes.core.k8s` với `state: present`, KHÔNG dump nội dung Secret ra stdout (set `no_log: true` cho task đó).

## Lint & test

- `.ansible-lint` cấu hình profile `production`:
  ```yaml
  profile: production
  skip_list: []
  ```
- Mỗi role có README.md tối thiểu liệt kê: mục đích, biến đầu vào (defaults + required), dependencies, ví dụ chạy.
- Ưu tiên `ansible.builtin.*` FQCN cho mọi module để lint qua profile production.

## Tags

Mỗi role gắn tag tên role để chạy chọn lọc:
```yaml
- name: Configure etcd
  ansible.builtin.template: { src: etcd.conf.yml.j2, dest: /etc/etcd/etcd.conf.yml, ... }
  notify: restart etcd
  tags: [etcd, etcd-config]
```

Tags tối thiểu mỗi role: `<role>`, `<role>-install`, `<role>-config`.

## Versioning của file template

Templates Jinja2 đặt header chuẩn:
```
# {{ ansible_managed }}
# Generated for cluster {{ cluster_name }} - DO NOT EDIT MANUALLY
```

## Disk layout convention

Mỗi VM có 2 phân vùng: OS (`/`) và data (`/u01`). **Mọi data dịch vụ đều nằm dưới `/u01`.**

| Dịch vụ | Đường dẫn |
|---|---|
| etcd data | `/u01/etcd` |
| PostgreSQL PGDATA | `/u01/postgresql/16/main` |

Role `os_prep` phải verify `/u01` đã mount trước khi tạo thư mục con (dùng `ansible.builtin.stat` + `ansible.builtin.fail`).
Không tạo thư mục data trực tiếp dưới `/var/lib` hay `/` — vi phạm convention này là bug.

## Proxy support (corporate environments)

Dự án hỗ trợ deploy qua HTTP proxy bằng cách **không** ghi proxy vào system-wide mà dùng **per-task proxy** để kiểm soát chính xác.

### Biến proxy (group_vars/all/main.yml)

```yaml
http_proxy: ""          # rỗng = không dùng proxy
https_proxy: ""
no_proxy: "localhost,127.0.0.1,{{ network_cidr }}"
proxy_env:              # dict dùng chung — không override trực tiếp
  http_proxy: "{{ http_proxy }}"
  https_proxy: "{{ https_proxy if https_proxy else http_proxy }}"
  no_proxy: "{{ no_proxy }}"
```

### Pattern bắt buộc

**Task fetch từ internet** (get_url, pip, uri → GitHub, PyPI, apt repo…) phải có:
```yaml
environment: "{{ proxy_env }}"
```

**Task nội bộ cluster** (etcdctl, patronictl, psql, curl Patroni REST…) **không** thêm `environment` — tránh proxy internal traffic qua proxy server.

### Cấu hình apt proxy

`os_prep` tự động quản lý `/etc/apt/apt.conf.d/95ansible-proxy`:
- Khi `http_proxy` non-empty: ghi file (template `apt-proxy.conf.j2`).
- Khi `http_proxy` rỗng: xóa file (idempotent).

### Tham chiếu theo phase

| Phase | Task cần `environment: "{{ proxy_env }}"` |
|---|---|
| Phase 3 (etcd) | `get_url` download binary từ GitHub |
| Phase 4 (patroni) | `pip` install vào venv |
| Phase 5 (K8s) | K8s worker pull image — ngoài scope bundle, cần containerd proxy |

## Không làm những điều sau

- Không hardcode IP/hostname trong template — luôn dùng `hostvars[item].ansible_host`.
- Không sửa `postgresql.conf` trực tiếp — Patroni sở hữu nó.
- Không tạo K8s resource bằng `command: kubectl apply` — dùng `kubernetes.core.k8s`.
- Không `pip install -g` — dùng OS package hoặc venv.
- Không commit `.vault_pass`, `*.retry`, `__pycache__`.
