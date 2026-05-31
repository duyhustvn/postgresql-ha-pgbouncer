# Role: os_prep

OS baseline preparation for all PostgreSQL HA nodes.

## Purpose

Applies OS-level requirements before any service is installed:
packages, system users, sysctl tuning, huge pages, and kernel watchdog.

**Not managed by this role** (handled externally):
- Firewall rules — managed via iptables directly on each node
- Time synchronization — managed via existing infrastructure

## Variables

### defaults/main.yml (overridable)

| Variable | Default | Description |
|---|---|---|
| `os_prep_data_mount` | `/u01` | Data partition mount point used for pre-flight check |
| `os_prep_verify_mount` | `true` | Fail if `/u01` is not a separate mountpoint. Set `false` in test/dev |
| `os_prep_required_packages` | see defaults | Base packages to install via apt |
| `etcd_data_dir` | `/u01/etcd` | etcd data directory (created with `etcd:etcd` ownership) |
| `postgresql_data_dir` | `/u01/postgresql` | PostgreSQL data root (created with `postgres:postgres` ownership) |
| `vm_nr_hugepages` | `0` (disabled) | Number of 2MB huge pages. Set `8400` for 16GB shared_buffers |
| `vm_overcommit_memory` | `2` | OOM protection — never overcommit |
| `vm_swappiness` | `1` | Minimize swapping |
| `net_tcp_keepalive_time` | `60` | TCP keepalive interval (seconds) |
| `kernel_shmmax` | `17179869184` | SysV shmmax in bytes (16GiB default) |
| `kernel_shmall` | `4194304` | SysV shmall pages (shmmax / 4096) |
| `os_prep_sysctl_file` | `/etc/sysctl.d/99-ansible-os-prep.conf` | Persisted sysctl file |

### Required (no default — must be set in inventory)

None.

## Production tuning

For a production server with 64GB RAM and `shared_buffers = 16GB`:

```yaml
# group_vars/db_nodes/main.yml
vm_nr_hugepages: 8400   # get exact value: postgres -C shared_memory_size_in_huge_pages
kernel_shmmax: 17179869184
kernel_shmall: 4194304
```

## Watchdog

Patroni uses the kernel watchdog device (`/dev/watchdog`) as a fencing mechanism to prevent split-brain.

**How it works:**

1. Patroni continuously writes to `/dev/watchdog` while it is healthy.
2. If Patroni hangs or crashes and stops writing, the kernel watchdog timer expires and **forces an immediate reboot** of the node — no human intervention required.
3. After reboot, the node rejoins the cluster as a replica, leaving only one primary.

**Why this matters:**

Without a watchdog, a network partition could leave a former primary running in isolation, still accepting writes, while a new primary is elected on the other side. Both nodes would write simultaneously, corrupting replication (split-brain). The watchdog ensures the isolated primary self-destructs before that can happen.

**What this role configures:**

- Loads the `softdog` kernel module (software watchdog emulation, no extra hardware needed).
- Persists `softdog` in `/etc/modules-load.d/` so it survives reboots.
- Creates a udev rule granting the `postgres` user ownership of `/dev/watchdog`, allowing Patroni (which runs as `postgres`) to open the device.

**Patroni side** (configured in the `patroni` role):

```yaml
watchdog:
  mode: required   # Patroni refuses to start if /dev/watchdog is unavailable
  device: /dev/watchdog
  safety_margin: 5
```

## Dependencies

None.

## Example

```yaml
# Run full role
ansible-playbook playbooks/os_prep.yml

# Run only sysctl tuning
ansible-playbook playbooks/os_prep.yml --tags os_prep-sysctl

# Skip mount check (test/dev environment)
ansible-playbook playbooks/os_prep.yml -e os_prep_verify_mount=false

# Dry run
ansible-playbook playbooks/os_prep.yml --check --diff
```

## Tags

| Tag | Scope |
|---|---|
| `os_prep` | All tasks |
| `os_prep-install` | Package installation |
| `os_prep-config` | Users, sysctl, hugepages, watchdog |
| `os_prep-sysctl` | Only sysctl parameters |
| `os_prep-hugepages` | Only huge pages |
| `os_prep-watchdog` | Only softdog / udev rule |
