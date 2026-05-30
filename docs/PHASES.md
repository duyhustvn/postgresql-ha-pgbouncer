# PHASES.md — Index 6 phase

Thứ tự bắt buộc; mỗi phase có acceptance criteria phải đạt trước khi sang phase sau.

## Sơ đồ phụ thuộc

```
Phase 1  Repo scaffold + inventory model         (skeleton, no logic)
  │
Phase 2  OS prep role                            (chuẩn bị 3 PG node)
  │
Phase 3  etcd cluster role                       (3-node DCS up & healthy)
  │
Phase 4  PostgreSQL + Patroni role               (1 primary + 2 replica streaming)
  ├── 4a: skeleton roles + templates
  └── 4b: orchestration + bootstrap logic
  │
Phase 5  K8s frontend tier                       (HAProxy + PgBouncer Deployments)
  │
Phase 6  Verification + Day-2 playbooks          (smoke tests, switchover, reinit)
```

## Tổng quan từng phase

### Phase 1 — Scaffold
Sinh khung repo Ansible: `ansible.cfg`, `requirements.yml`, layout `inventories/prod/`, role skeleton trống cho 6 role (`os_prep`, `etcd`, `postgresql`, `patroni`, `haproxy`, `pgbouncer`), playbook stubs gọi role. Không có business logic. Lint pass.

**Vì sao tách:** Tránh việc tạo skeleton lẫn lộn với logic ở những phase phức tạp.

### Phase 2 — OS prep role
Implement `roles/os_prep`: cài package nền, tạo user `postgres`, áp sysctl (overcommit, swappiness, nr_hugepages), load `softdog` module, mount tmpfs nếu cần, mở firewall các cổng (5432, 8008, 2379, 2380).

### Phase 3 — etcd
Implement `roles/etcd`: tải binary, sinh `etcd.conf.yml` per-node, systemd unit, bootstrap 3-node cluster mode `new`. Verify `etcdctl endpoint health` đủ 3 healthy.

### Phase 4 — PostgreSQL + Patroni
- **4a (skeleton):** roles `postgresql` (chỉ cài binary, KHÔNG initdb — Patroni làm) + `patroni` (cài binary, systemd unit, template patroni.yml). Templates rỗng có placeholder.
- **4b (logic):** điền template patroni.yml đầy đủ, orchestration để bootstrap leader trước rồi replicas join, handler restart/reload an toàn, tạo user replication/rewind/dify.

### Phase 5 — K8s frontend
Implement `roles/haproxy` + `roles/pgbouncer` dùng `kubernetes.core.k8s`. Sinh manifests qua Jinja2 (Namespace, ConfigMap, Secret, Deployment, Service, NetworkPolicy). PgBouncer Secret chứa `userlist.txt` với SCRAM verifier query động từ PG primary.

### Phase 6 — Verification + Day-2
- `playbooks/verify.yml`: chạy smoke tests qua mỗi tầng (etcdctl health, patronictl list, kubectl get pods, psql qua PgBouncer Service `SELECT 1`).
- `playbooks/switchover.yml`: thực hiện `patronictl switchover` có xác nhận.
- `playbooks/reinit_replica.yml`: reinit replica lag/hỏng.

## Mỗi phase có gì trong file của nó

Mỗi `phases/PHASE-N-*.md` có cấu trúc thống nhất:
- **Mục tiêu**
- **In scope / Out of scope**
- **Deliverables** (đường dẫn file cụ thể)
- **Context files cần load** (để paste vào prompt)
- **Implementation notes** (gợi ý kỹ thuật, ràng buộc)
- **Acceptance criteria** (test phải pass)

Prompt sẵn dùng nằm tập trung ở `PROMPTS.md`, mỗi prompt tham chiếu đúng phase + skill cần.

## Quy ước "Done" giữa các phase

Trước khi prompt phase tiếp theo, chạy:
```
ansible-lint
ansible-playbook playbooks/<phase>.yml --check --diff
ansible-playbook playbooks/<phase>.yml          # apply thật
ansible-playbook playbooks/<phase>.yml --check  # phải KHÔNG có changed
```

Phase được coi là Done khi: lint sạch, apply thật xong, chạy lại check không có changed (idempotency confirmed).
