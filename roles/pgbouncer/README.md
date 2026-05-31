# Role: pgbouncer

Deploy PgBouncer to Kubernetes in transaction-mode connection pooling.

## Purpose

Applies a Kubernetes Deployment (3 replicas) and ClusterIP Service for PgBouncer.
PgBouncer runs in transaction mode, multiplexing up to 2000 client connections per
pod down to ≤50 server connections via HAProxy to the PostgreSQL primary.

Connection formula (must hold):
`pgbouncer_replicas × (default_pool_size + reserve_pool_size) ≤ pg_max_connections × 0.85`

Default: `3 × (50 + 10) = 180 ≤ 200 × 0.85 = 170` — adjust `pg_max_connections` upward
or reduce pool sizes if this constraint is violated.

## Resources created

| Kind | Name | Description |
|---|---|---|
| Namespace | `db` | Shared with haproxy |
| ConfigMap | `pgbouncer-config` | Rendered `pgbouncer.ini` |
| Secret | `pgbouncer-userlist` | `userlist.txt` with SCRAM-SHA-256 verifiers |
| Deployment | `pgbouncer` | 3 replicas, podAntiAffinity |
| Service | `pgbouncer` | ClusterIP, port 6432 |
| PodDisruptionBudget | `pgbouncer` | minAvailable: 2 |

## Prerequisites

1. **Phase 4 complete**: users `dify` and `pgbouncer_admin` must exist in PostgreSQL
   with SCRAM-SHA-256 passwords before this role runs.
2. **psycopg2 on controller**: `community.postgresql.postgresql_query` requires
   `python3-psycopg2` (or `psycopg2-binary`) on the Ansible controller.
   Install: `pip install psycopg2-binary`
3. **Patroni primary reachable**: the controller must reach `{{ postgresql_port }}`
   on the primary's `ansible_host` IP. If a firewall blocks this, use an SSH tunnel
   and set `login_host` / `login_port` accordingly.
4. **kubeconfig**: `~/.kube/config` or `KUBECONFIG` env var pointing to the target cluster.

## Secret security note

`pgbouncer-userlist` Secret holds SCRAM verifiers (hashed, not plaintext passwords).
These are stored base64-encoded in Kubernetes etcd — any user with `get secret`
permission can decode them. For production, consider SealedSecrets or Vault.

## Variables

### defaults/main.yml (overridable)

| Variable | Default | Description |
|---|---|---|
| `pgbouncer_replicas` | `3` | Number of K8s replicas |
| `pgbouncer_port` | `6432` | PgBouncer listen port |
| `pgbouncer_image` | `edoburu/pgbouncer:1.23` | Container image |
| `pgbouncer_pool_mode` | `transaction` | Pool mode (do not change to session) |
| `pgbouncer_max_client_conn` | `2000` | Max client connections per pod |
| `pgbouncer_default_pool_size` | `50` | Server connections per pool |
| `pgbouncer_min_pool_size` | `10` | Minimum server connections kept open |
| `pgbouncer_reserve_pool_size` | `10` | Reserve connections per pool |
| `pgbouncer_reserve_pool_timeout` | `3` | Seconds before using reserve pool |
| `pgbouncer_max_db_connections` | `100` | Max total server connections per database |
| `pgbouncer_server_lifetime` | `600` | Server connection lifetime (recycle after failover) |
| `pgbouncer_server_idle_timeout` | `120` | Close idle server connections after N seconds |
| `pgbouncer_server_connect_timeout` | `15` | Timeout connecting to PostgreSQL |
| `pgbouncer_query_wait_timeout` | `120` | Max seconds a query can wait for a connection |

### Required from group_vars/all (via vault)

| Variable | Description |
|---|---|
| `postgres_superuser_password` | Used to query SCRAM verifiers from `pg_authid` |
| `k8s_namespace` | Kubernetes namespace (`db`) |
| `cluster_name` | Cluster label (`postgres-ha`) |
| `haproxy_frontend_port` | HAProxy service port for the `[databases]` section |

## Dependencies

- Role `haproxy` (declared in `meta/main.yml`): the `pg-haproxy` Service must exist
  before PgBouncer pods start, so they can resolve `pg-haproxy.db.svc.cluster.local`.
- `patroni_primary_host` fact: set automatically by `find_primary.yml` (included in
  `tasks/main.yml`). No manual action required.

## Example

```yaml
- name: Deploy PgBouncer
  hosts: k8s_admin
  roles:
    - role: pgbouncer
```

## Tags

| Tag | Scope |
|---|---|
| `pgbouncer` | All tasks |
| `pgbouncer-install` | Deployment + Service + PDB apply + rollout wait |
| `pgbouncer-config` | SCRAM fetch, ConfigMap + Secret render + apply |
