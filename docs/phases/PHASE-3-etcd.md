# PHASE-3 — etcd Cluster

## Mục tiêu

Implement `roles/etcd` để cài 3-node etcd cluster làm DCS cho Patroni. Cuối phase: 3 endpoint healthy, 1 leader.

## In scope

- Download binary etcd từ GitHub release, version pin (`etcd_version`).
- Cài binary vào `/usr/local/bin/etcd` và `/usr/local/bin/etcdctl`.
- Tạo data dir `/var/lib/etcd` owner `etcd:etcd`, mode 0700.
- Sinh config `/etc/etcd/etcd.conf.yml` từ template, các giá trị động lấy từ `groups['etcd']` và `hostvars`.
- Sinh systemd unit `/etc/systemd/system/etcd.service`.
- Bootstrap 3 node song song (initial-cluster-state = `new`).
- Verify cluster health bằng `etcdctl endpoint health`.

## Out of scope

- TLS giữa các node (HTTP đủ cho prod nội bộ; có thể nâng cấp ở day-2).
- Snapshot backup cronjob — day-2.
- Adding/removing node — edge case.

## Deliverables

```
roles/etcd/
├── tasks/
│   ├── main.yml
│   ├── install.yml          # download + install binary
│   ├── configure.yml        # template config + systemd unit
│   ├── start.yml            # enable + start, đợi healthy
│   └── verify.yml           # etcdctl endpoint health
├── templates/
│   ├── etcd.conf.yml.j2
│   └── etcd.service.j2
├── defaults/main.yml        # etcd_version, paths
├── handlers/main.yml        # restart etcd (serial)
├── meta/main.yml
└── README.md
```

```
playbooks/etcd.yml            # gọi role etcd cho group etcd
```

## Context files cần load

- `docs/CLAUDE.md`
- `docs/ARCHITECTURE.md`
- `docs/skills/project-conventions.md`
- `docs/skills/etcd-spec.md`
- `docs/phases/PHASE-3-etcd.md` (file này)

## Implementation notes

- Bootstrap pattern (quan trọng để đúng):
  ```yaml
  - hosts: etcd
    tasks:
      - import_role: { name: etcd, tasks_from: install }
      - import_role: { name: etcd, tasks_from: configure }
      - import_role: { name: etcd, tasks_from: start }     # parallel start

  - hosts: etcd
    run_once: true
    tasks:
      - import_role: { name: etcd, tasks_from: verify }
  ```
- Khi đã bootstrap, lần chạy thứ 2 KHÔNG được sửa `initial-cluster-state` (nếu vẫn `new` thì etcd vẫn ok vì check sau bootstrap). Nếu Claude Code muốn detect "đã bootstrap" → check sự tồn tại của `/var/lib/etcd/member/` thư mục, nếu có thì set var `etcd_bootstrapped: true` và skip task bootstrap-only.
- Handler `restart etcd` phải dùng `listen` + chạy trên từng host tuần tự, KHÔNG đồng thời (sẽ làm sập quorum):
  ```yaml
  # Trong playbook gọi handler, set `serial: 1`
  ```
- `etcdctl` cần env `ETCDCTL_API=3` (mặc định ở v3.4+).
- Health verify task dùng `run_once: true` + `delegate_to: first_etcd_host`, retry 12 lần / 5s.

## Acceptance criteria

1. `ansible-playbook playbooks/etcd.yml` chạy lần đầu thành công.
2. SSH vào pg1: `etcdctl --endpoints=http://10.0.0.1:2379,http://10.0.0.2:2379,http://10.0.0.3:2379 endpoint health` ra 3 dòng `healthy`.
3. `etcdctl ... endpoint status -w table` cho thấy 1 node có `IS_LEADER: true`.
4. Chạy lại playbook → KHÔNG có changed.
5. Kill 1 etcd (`systemctl stop etcd` trên pg2), cluster vẫn healthy với 2/3, leader vẫn tồn tại (có thể bị re-elect). Bật lại → quay về 3/3.
6. `ansible-lint` clean.

## Cảnh báo

Nếu chỉ 2/3 etcd lên sau khi bootstrap, **dừng** — phase 4 không nên tiếp tục với etcd không khỏe, Patroni sẽ flap.
