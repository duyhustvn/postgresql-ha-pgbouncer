# Role: pgbouncer

Deploy PgBouncer to Kubernetes in transaction-mode connection pooling.

## Purpose

Applies a Kubernetes Deployment (3 replicas) and ClusterIP Service for PgBouncer.
PgBouncer runs in transaction mode, multiplexing up to 2000 client connections per
pod down to ≤50 server connections via HAProxy to the PostgreSQL primary.

Connection formula (must hold):
`pgbouncer_replicas × (default_pool_size + reserve_pool_size) ≤ pg_max_connections × 0.85`

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
| `pgbouncer_reserve_pool_size` | `10` | Reserve connections per pool |
| `pgbouncer_server_lifetime` | `600` | Server connection lifetime (recycle after failover) |

### Required from group_vars/all (via vault)

| Variable | Description |
|---|---|
| `dify_db_password` | Dify application database password |
| `pgbouncer_admin_password` | PgBouncer admin user password |
| `k8s_namespace` | Kubernetes namespace (`db`) |

## Dependencies

- `haproxy` (service must exist for PgBouncer to connect to)

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
| `pgbouncer-install` | K8s Deployment + Service apply |
| `pgbouncer-config` | K8s Secret (pgbouncer.ini + userlist) |
