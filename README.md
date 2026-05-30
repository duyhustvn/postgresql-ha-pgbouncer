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
