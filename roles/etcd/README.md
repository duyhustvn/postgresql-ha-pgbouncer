# Role: etcd

Deploy and configure an etcd cluster used as DCS (Distributed Configuration Store) for Patroni.

## Purpose

Downloads etcd binary, creates system user, writes etcd config, starts the service,
and bootstraps a 3-node cluster. All data goes under `/u01/etcd`.

## Variables

### defaults/main.yml (overridable)

| Variable | Default | Description |
|---|---|---|
| `etcd_user` | `etcd` | System user running etcd |
| `etcd_group` | `etcd` | System group |
| `etcd_install_dir` | `/usr/local/bin` | Binary installation path |
| `etcd_data_dir` | `/u01/etcd` | Data directory |
| `etcd_config_dir` | `/etc/etcd` | Config directory |
| `etcd_client_port` | `2379` | Client API port |
| `etcd_peer_port` | `2380` | Peer communication port |

### group_vars/etcd/main.yml

| Variable | Description |
|---|---|
| `etcd_cluster_token` | Unique cluster token (default: `dify-pg-etcd`) |
| `etcd_initial_cluster_state` | `new` on first boot, `existing` on rejoin |

### Required from group_vars/all

| Variable | Description |
|---|---|
| `etcd_version` | etcd binary version to download (e.g. `v3.5.30`) |
| `cluster_name` | Used in cluster token and naming |

## Dependencies

- `os_prep` (applied first)

## Example

```yaml
- name: Deploy etcd
  hosts: etcd
  become: true
  roles:
    - role: etcd
```

## Tags

| Tag | Scope |
|---|---|
| `etcd` | All tasks |
| `etcd-install` | Binary download and service setup |
| `etcd-config` | Config file generation |
