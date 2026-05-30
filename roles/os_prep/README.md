# Role: os_prep

OS baseline preparation for all PostgreSQL HA nodes.

## Purpose

Applies OS-level requirements before any service is installed: packages, sysctl
tuning, user/group creation, and ensuring `/u01` (data partition) is accessible.

## Variables

### defaults/main.yml (overridable)

| Variable | Default | Description |
|---|---|---|
| `os_prep_data_mount` | `/u01` | Mount point for the data partition |
| `os_prep_required_packages` | `[python3-psycopg2, acl, curl, ca-certificates]` | Extra packages to ensure installed |

### Required (no default — must be set in inventory)

None at this phase.

## Dependencies

None.

## Example

```yaml
- name: OS baseline
  hosts: db_nodes
  become: true
  roles:
    - role: os_prep
```

## Tags

| Tag | Scope |
|---|---|
| `os_prep` | All tasks |
| `os_prep-install` | Package installation |
| `os_prep-config` | sysctl / limits / directory setup |
