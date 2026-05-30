# PHASE-2 — OS Preparation Role

## Mục tiêu

Implement `roles/os_prep` để chuẩn bị 3 PG node sẵn sàng cho etcd + Patroni: gói nền, user, sysctl chống OOM, huge pages, kernel watchdog, firewall.

## In scope

- Cài base packages: `curl`, `gnupg`, `python3`, `python3-psycopg2`, `python3-pip`, `acl`, `chrony`.
- Tạo user `postgres` và `etcd` (system, no login).
- Apply sysctl:
  - `vm.overcommit_memory = 2`
  - `vm.swappiness = 1`
  - `vm.nr_hugepages = {{ vm_nr_hugepages | default(8400) }}` (cho shared_buffers 16GB).
  - `net.ipv4.tcp_keepalive_time = 60`
  - `kernel.shmmax`, `kernel.shmall` đủ cho shared_buffers.
- Load kernel module `softdog` (Patroni watchdog), persist trong `/etc/modules-load.d/softdog.conf`.
- Verify `/u01` đã mount (fail fast nếu chưa — tránh ghi nhầm vào OS partition).
- Tạo thư mục data dưới `/u01`: `/u01/etcd`, `/u01/postgresql` (biến `etcd_data_dir` và `postgresql_data_dir`, mặc định như trên).
- Mở firewall (ufw hoặc nftables tùy distro) cổng: 5432, 8008, 2379, 2380 — chỉ allow từ `network_cidr`. Skip nếu firewall không active.
- Sync time qua chrony, đảm bảo offset < 100ms (Patroni leader election nhạy với clock skew).

## Out of scope

- Cài PostgreSQL, etcd, Patroni — phase 3 và 4.
- Tạo replication user — Patroni làm.
- Tinkering K8s node — node K8s đã có sẵn.

## Deliverables

```
roles/os_prep/
├── tasks/
│   ├── main.yml             # import_tasks điều phối
│   ├── packages.yml
│   ├── users.yml
│   ├── sysctl.yml
│   ├── hugepages.yml
│   ├── watchdog.yml
│   ├── firewall.yml
│   └── time_sync.yml
├── defaults/main.yml        # vm_nr_hugepages, network_cidr, etcd_data_dir, postgresql_data_dir, …
├── handlers/main.yml        # reload sysctl, restart chrony
├── meta/main.yml
└── README.md
```

```
playbooks/os_prep.yml         # gọi role os_prep cho group db_nodes
```

## Context files cần load

- `docs/CLAUDE.md`
- `docs/ARCHITECTURE.md`
- `docs/skills/project-conventions.md`
- `docs/phases/PHASE-2-os-prep.md` (file này)

## Implementation notes

- Dùng `ansible.posix.sysctl` với `reload: yes` cho từng giá trị, hoặc `sysctl_set: yes` để persist.
- Huge pages: tính số trang cần qua biến `vm_nr_hugepages`. Cho `shared_buffers = 16GB` và page size 2MB → cần ~8400 trang (16384/2 + overhead). Documentation note: sau khi PG cài, lấy số chính xác từ `postgres -C shared_memory_size_in_huge_pages`.
- Watchdog: 
  ```yaml
  - name: Ensure softdog loaded
    community.general.modprobe: { name: softdog, state: present }
  - name: Persist softdog at boot
    ansible.builtin.copy:
      content: "softdog\n"
      dest: /etc/modules-load.d/softdog.conf
      mode: '0644'
  ```
- Firewall: detect `ufw` hay `firewalld` qua `ansible_facts['service_mgr']` hoặc kiểm `which ufw`. Skip nếu không có. Mở cổng từ `network_cidr` thôi, không `0.0.0.0/0`.
- Time sync: cài `chrony`, enable service. Verify với `chronyc tracking` (`changed_when: false`).
- Verify `/u01` mount: dùng `ansible.builtin.stat` kiểm tra `mountpoint: /u01`, fail với `ansible.builtin.fail` nếu chưa mount — không tạo thư mục con nếu chưa có data partition.
- Mọi task có `tags: [os-prep, os-prep-<sub>]` (sysctl, hugepages, watchdog…).

## Acceptance criteria

### Từ Ansible controller

```bash
# 1. Chạy lần đầu — phải thành công, không error
ansible-playbook playbooks/os_prep.yml

# 2. Chạy lại lần 2 — không có task nào "changed"
ansible-playbook playbooks/os_prep.yml

# 3. Dry-run phải chạy được không lỗi
ansible-playbook playbooks/os_prep.yml --check --diff

# 4. Lint sạch
ansible-lint roles/os_prep/

# 5. Verify từ xa qua ad-hoc (thay thế SSH tay, chạy trên cả 3 node)
ansible db_nodes -b -m command -a "sysctl vm.overcommit_memory vm.swappiness net.ipv4.tcp_keepalive_time"
ansible db_nodes -b -m command -a "lsmod | grep softdog"
ansible db_nodes -b -m command -a "ls -la /dev/watchdog"
ansible db_nodes -b -m command -a "id postgres; id etcd"
ansible db_nodes -b -m stat -a "path=/u01/etcd"
ansible db_nodes -b -m stat -a "path=/u01/postgresql"
ansible db_nodes -b -m command -a "cat /etc/sysctl.d/99-ansible-os-prep.conf"
ansible db_nodes -b -m command -a "cat /etc/modules-load.d/softdog.conf"
```

### Kết quả mong đợi

| Lệnh verify | Kết quả mong đợi |
|---|---|
| `sysctl vm.overcommit_memory` | `vm.overcommit_memory = 2` |
| `sysctl vm.swappiness` | `vm.swappiness = 1` |
| `sysctl net.ipv4.tcp_keepalive_time` | `net.ipv4.tcp_keepalive_time = 60` |
| `lsmod \| grep softdog` | dòng chứa `softdog` |
| `ls -la /dev/watchdog` | owner `postgres`, mode `600` |
| `id postgres` | system user tồn tại |
| `id etcd` | system user tồn tại |
| `stat /u01/etcd` | Uid: `etcd`, mode `0700` |
| `stat /u01/postgresql` | Uid: `postgres`, mode `0700` |

### Lưu ý môi trường test

Nếu chạy trên VM không có `/u01` mount riêng, thêm flag:

```bash
ansible-playbook playbooks/os_prep.yml -e os_prep_verify_mount=false
```

Nếu VM nhỏ (< 16GB RAM), giữ `os_prep_vm_nr_hugepages: 0` trong group_vars — huge pages task tự skip.
