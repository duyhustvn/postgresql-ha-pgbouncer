# postgresql-ha-pgbouncer

Ansible project deploying a **PostgreSQL 16 HA cluster** for [Dify](https://dify.ai), with
full connection-pooling via PgBouncer and smart routing via HAProxy.

## Architecture

```
Dify pods  →  PgBouncer (K8s, 3 replica, port 6432)
           →  HAProxy   (K8s, 2 replica, port 5000, httpchk /primary)
           →  3 × PostgreSQL 16 + Patroni + etcd  (bare-metal / VMs)
```

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for full topology and sizing rationale.

## Prerequisites

- Ansible >= 2.16 on the controller
- Python 3.10+ on target nodes
- Target OS: Ubuntu 22.04 LTS
- A running Kubernetes cluster (namespace `db` will be created)

## Quick start

```bash
# 1. Install required collections
ansible-galaxy collection install -r requirements.yml

# 2. Create vault password file (gitignored)
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass

# 3. Edit inventory with real IPs / passwords
ansible-vault edit inventories/prod/group_vars/all/vault.yml

# 4. Full deployment
ansible-playbook playbooks/site.yml

# 5. Verify cluster
ansible-playbook playbooks/verify.yml
```

## Playbooks

| Playbook | Purpose |
|---|---|
| `site.yml` | Full deployment (all roles in order) |
| `os_prep.yml` | OS baseline (packages, kernel params, users) |
| `etcd.yml` | etcd cluster on the 3 PG nodes |
| `database.yml` | PostgreSQL 16 + Patroni |
| `k8s_frontend.yml` | HAProxy + PgBouncer K8s Deployments |
| `verify.yml` | Smoke-test cluster health |

## PostgreSQL Users

### Auto-created by Patroni bootstrap

Patroni tạo các user này tự động khi bootstrap cluster (Stage 2 của `database.yml`). Không cần làm gì thêm.

| User | Quyền | Password variable |
|---|---|---|
| `postgres` | Superuser | `vault_postgres_superuser_password` |
| `admin` | `CREATEROLE CREATEDB` | `vault_postgres_superuser_password` |
| `replicator` | Streaming replication | `vault_postgres_replication_password` |
| `rewind_user` | `pg_rewind` | `vault_postgres_rewind_password` |

### Cần tạo thủ công

Các user này phải tạo sau khi cluster đã chạy. Kết nối vào primary node và chạy:

```bash
sudo -u postgres psql
```

**`pgbouncer_admin`** — bắt buộc nếu dùng PgBouncer admin/stats console:

```sql
CREATE ROLE pgbouncer_admin
  WITH LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE
  PASSWORD 'your-password';
```

Sau khi tạo, thêm vào inventory và chạy lại `k8s_frontend.yml`:

```yaml
# inventories/prod/group_vars/all/main.yml
pgbouncer_auth_users:
  - pgbouncer_admin
```

### Application users

Thêm application user vào Stage 4 của `database.yml`, khai báo database và user trong inventory:

```yaml
# inventories/prod/group_vars/all/main.yml
pgbouncer_databases:
  - { name: myapp, dbname: myapp }

pgbouncer_auth_users:
  - pgbouncer_admin
  - myapp_user
```

Task mẫu tạo application user + database:

```yaml
- name: Create application role
  community.postgresql.postgresql_user:
    name: myapp_user
    password: "{{ myapp_db_password }}"
    role_attr_flags: LOGIN,NOSUPERUSER,NOCREATEDB,NOCREATEROLE
    login_user: postgres
    login_unix_socket: /var/run/postgresql
    state: present
  become_user: postgres
  delegate_to: "{{ patroni_primary_inventory_host }}"
  run_once: true
  no_log: true

- name: Create application database
  community.postgresql.postgresql_db:
    name: myapp
    owner: myapp_user
    encoding: UTF8
    lc_collate: "en_US.UTF-8"
    lc_ctype: "en_US.UTF-8"
    template: template0
    login_user: postgres
    login_unix_socket: /var/run/postgresql
    state: present
  become_user: postgres
  delegate_to: "{{ patroni_primary_inventory_host }}"
  run_once: true
```

## Day-2 operations

```bash
# Planned primary switchover
ansible-playbook playbooks/switchover.yml

# Re-init a diverged replica
ansible-playbook playbooks/reinit_replica.yml
```

## Documentation

- [Architecture](docs/ARCHITECTURE.md)
- [Phase plan](docs/PHASES.md)
- [Component specs](docs/skills/)
