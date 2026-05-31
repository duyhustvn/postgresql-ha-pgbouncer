# Role: haproxy

Deploy HAProxy to Kubernetes as the smart routing layer to the PostgreSQL primary.

## Purpose

Applies a Kubernetes Deployment (2 replicas) and ClusterIP Service for HAProxy.
HAProxy uses `httpchk` against Patroni's REST endpoint `/:8008/primary` — only
the current primary returns HTTP 200, so HAProxy marks replicas DOWN automatically
on failover. The annotation `checksum/config` on the Deployment triggers a rolling
restart whenever `haproxy.cfg` changes.

## Resources created

| Kind | Name | Description |
|---|---|---|
| Namespace | `db` | Shared with pgbouncer |
| ConfigMap | `pg-haproxy-config` | Rendered `haproxy.cfg` |
| Deployment | `pg-haproxy` | 2 replicas, podAntiAffinity |
| Service | `pg-haproxy` | ClusterIP, port 5000 |
| NetworkPolicy | `pg-haproxy-allow-pgbouncer` | Allow ingress from pgbouncer pods only |
| PodDisruptionBudget | `pg-haproxy` | minAvailable: 1 |

## Variables

### defaults/main.yml (overridable)

| Variable | Default | Description |
|---|---|---|
| `haproxy_replicas` | `2` | Number of K8s replicas |
| `haproxy_frontend_port` | `5000` | PostgreSQL frontend port |
| `haproxy_stats_port` | `7000` | HAProxy stats page port |
| `haproxy_image` | `haproxy:2.9` | Container image |
| `haproxy_backend_maxconn` | `200` | Max TCP sessions per backend server |

### Required from group_vars/all

| Variable | Description |
|---|---|
| `k8s_namespace` | Kubernetes namespace (`db`) |
| `cluster_name` | Cluster label (`postgres-ha`) |
| `patroni_rest_port` | Patroni health-check port (`8008`) |
| `postgresql_port` | PostgreSQL backend port (`5432`) |

## Dependencies

None. Runs on `k8s_admin` (localhost) with kubeconfig access.

## NetworkPolicy note

The `pg-haproxy-allow-pgbouncer` NetworkPolicy requires a CNI that enforces
NetworkPolicy (Calico, Cilium). With Flannel (default CNI), the manifest is
applied but has no enforcement effect. Both HAProxy and PgBouncer pods must be
in the same `db` namespace (they are).

## Example

```yaml
- name: Deploy HAProxy
  hosts: k8s_admin
  roles:
    - role: haproxy
```

## Tags

| Tag | Scope |
|---|---|
| `haproxy` | All tasks |
| `haproxy-install` | Deployment + Service + PDB apply + rollout wait |
| `haproxy-config` | ConfigMap render + apply, NetworkPolicy |
