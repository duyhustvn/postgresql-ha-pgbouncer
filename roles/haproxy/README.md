# Role: haproxy

Deploy HAProxy to Kubernetes as the smart routing layer to the PostgreSQL primary.

## Purpose

Applies a Kubernetes Deployment (2 replicas) and ClusterIP Service for HAProxy.
HAProxy uses `httpchk` against Patroni's REST endpoint `/:8008/primary` — only
the current primary returns HTTP 200, so HAProxy marks replicas DOWN automatically
on failover.

## Variables

### defaults/main.yml (overridable)

| Variable | Default | Description |
|---|---|---|
| `haproxy_replicas` | `2` | Number of K8s replicas |
| `haproxy_frontend_port` | `5000` | PostgreSQL frontend port |
| `haproxy_stats_port` | `7000` | HAProxy stats page port |
| `haproxy_image` | `haproxy:2.9` | Container image |

### Required from group_vars/all

| Variable | Description |
|---|---|
| `k8s_namespace` | Kubernetes namespace (`db`) |
| `patroni_rest_port` | Port for Patroni health check (`8008`) |

## Dependencies

None (runs against `k8s_admin` / localhost with K8s API access).

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
| `haproxy-install` | K8s Deployment + Service apply |
| `haproxy-config` | ConfigMap with haproxy.cfg |
