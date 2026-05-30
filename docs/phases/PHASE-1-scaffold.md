# PHASE-1 — Repo Scaffold

## Mục tiêu

Sinh khung repo Ansible **trống về business logic**, đủ để các phase sau implement vào. Cuối phase này, repo chạy được `ansible-lint` và `ansible-playbook --syntax-check` không lỗi.

## In scope

- `ansible.cfg`, `requirements.yml`, `.ansible-lint`, `.gitignore`
- Layout `inventories/prod/` với `hosts.yml` mẫu (placeholder IP), `group_vars/all/main.yml`, `group_vars/all/vault.yml` (encrypted với password mẫu — sẽ thay sau).
- 6 role skeleton: `os_prep`, `etcd`, `postgresql`, `patroni`, `haproxy`, `pgbouncer`. Mỗi role có cấu trúc `tasks/main.yml`, `defaults/main.yml`, `handlers/main.yml`, `meta/main.yml`, `README.md`. `tasks/main.yml` chỉ chứa task `debug` "TODO".
- 6 playbook stub: `site.yml`, `os_prep.yml`, `etcd.yml`, `database.yml`, `k8s_frontend.yml`, `verify.yml`. Stub chỉ gọi role tương ứng với `tags`.
- `README.md` ở root repo: mô tả repo, cách chạy, link tới `docs/`.

## Out of scope

- Bất kỳ implement task thật nào (cài package, sinh config). Để các phase 2-5.

## Deliverables (đường dẫn cụ thể)

```
./ansible.cfg
./requirements.yml
./.ansible-lint
./.gitignore
./README.md

./inventories/prod/hosts.yml
./inventories/prod/group_vars/all/main.yml
./inventories/prod/group_vars/all/vault.yml          # encrypted
./inventories/prod/group_vars/db_nodes/main.yml
./inventories/prod/group_vars/etcd/main.yml

./playbooks/site.yml
./playbooks/os_prep.yml
./playbooks/etcd.yml
./playbooks/database.yml
./playbooks/k8s_frontend.yml
./playbooks/verify.yml

./roles/os_prep/{tasks,defaults,handlers,meta}/main.yml
./roles/os_prep/README.md
./roles/etcd/{tasks,defaults,handlers,meta}/main.yml
./roles/etcd/README.md
./roles/postgresql/{tasks,defaults,handlers,meta}/main.yml
./roles/postgresql/README.md
./roles/patroni/{tasks,defaults,handlers,meta}/main.yml
./roles/patroni/README.md
./roles/haproxy/{tasks,defaults,handlers,meta}/main.yml
./roles/haproxy/README.md
./roles/pgbouncer/{tasks,defaults,handlers,meta}/main.yml
./roles/pgbouncer/README.md
```

## Context files cần load vào prompt

- `docs/CLAUDE.md`
- `docs/ARCHITECTURE.md`
- `docs/skills/project-conventions.md`
- `docs/phases/PHASE-1-scaffold.md` (file này)

KHÔNG load skill component-cụ-thể (etcd-spec, patroni-spec, …) vì phase 1 chưa cần.

## Implementation notes

- `ansible.cfg`:
  ```
  [defaults]
  inventory = inventories/prod/hosts.yml
  vault_password_file = .vault_pass
  roles_path = roles
  host_key_checking = False
  retry_files_enabled = False
  forks = 10
  
  [ssh_connection]
  pipelining = True
  ```
- `requirements.yml`:
  ```yaml
  collections:
    - { name: ansible.posix,         version: ">=1.5.4" }
    - { name: community.general,     version: ">=8.0.0" }
    - { name: community.postgresql,  version: ">=3.4.0" }
    - { name: kubernetes.core,       version: ">=3.0.1" }
  ```
- `.ansible-lint`:
  ```yaml
  profile: production
  exclude_paths:
    - docs/
  ```
- `.gitignore` phải có: `.vault_pass`, `*.retry`, `__pycache__/`, `.cache/`, `inventories/*/host_vars/*.local.yml`.

## Acceptance criteria

1. `cd <repo> && ansible-galaxy install -r requirements.yml` không lỗi.
2. `ansible-playbook --syntax-check playbooks/site.yml` pass.
3. `ansible-lint` không warning ở profile production.
4. `tree -L 3 -d` ra đúng layout trên.
5. Mỗi role README có ít nhất: mục đích, biến đầu vào, ví dụ chạy (placeholder OK).

Sau khi pass: commit "Phase 1: scaffold" trước khi sang Phase 2.
