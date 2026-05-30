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
- Tạo thư mục data NVMe nếu mount điểm tồn tại: `/var/lib/postgresql`, `/var/lib/etcd` (đề xuất 2 disk khác nhau — biến `pg_data_mount` và `etcd_data_mount`, mặc định `/var/lib`).
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
├── defaults/main.yml        # vm_nr_hugepages, network_cidr, …
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
- Mọi task có `tags: [os-prep, os-prep-<sub>]` (sysctl, hugepages, watchdog…).

## Acceptance criteria

1. `ansible-playbook playbooks/os_prep.yml` chạy thành công lần đầu.
2. Chạy lại lần 2 → KHÔNG có task `changed`.
3. SSH vào từng PG node, verify:
   - `sysctl vm.overcommit_memory` ra `2`.
   - `sysctl vm.nr_hugepages` ra giá trị template.
   - `lsmod | grep softdog` thấy module.
   - `ls /dev/watchdog*` thấy device.
   - `id postgres` và `id etcd` tồn tại.
4. `ansible-lint` clean.
5. `--check --diff` chạy được không lỗi (idempotency).
