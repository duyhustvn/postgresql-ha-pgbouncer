# PROMPTS.md — Prompt sẵn dùng cho Claude Code

Mỗi phase một (hoặc hai) prompt. Paste vào Claude Code đúng thứ tự, đợi pass acceptance criteria của phase đó rồi mới sang prompt tiếp theo.

**Quy ước trong prompt:**
- `@file` = lệnh Claude Code load file vào context. Adjust syntax theo phiên bản Claude Code bạn dùng (`/add`, `@`, hoặc copy nội dung trực tiếp).
- Không thêm context khác ngoài file đã liệt kê. Tiết kiệm context window = chất lượng output cao hơn.

---

## Phase 1 — Scaffold repo

```
Đọc các file:
@docs/CLAUDE.md
@docs/ARCHITECTURE.md
@docs/skills/project-conventions.md
@docs/phases/PHASE-1-scaffold.md

Nhiệm vụ: Implement đúng phần "Deliverables" trong PHASE-1-scaffold.md.

Yêu cầu:
- Tuân thủ tất cả convention trong project-conventions.md.
- KHÔNG implement business logic — chỉ scaffold + stub.
- Mọi role có README.md tối thiểu (mục đích, biến đầu vào, ví dụ).
- Kết thúc phải pass: `ansible-playbook --syntax-check playbooks/site.yml` và `ansible-lint`.

Hoàn thành xong, list ra:
1. Cây thư mục sinh ra (tree -L 3).
2. Output `ansible-lint` (phải clean).
3. Output `ansible-playbook --syntax-check playbooks/site.yml`.

Hỏi tôi nếu cần làm rõ trước khi sinh code. Đừng đoán giá trị cho IP, password.
```

---

## Phase 2 — OS prep

```
Đọc các file:
@docs/CLAUDE.md
@docs/ARCHITECTURE.md
@docs/skills/project-conventions.md
@docs/phases/PHASE-2-os-prep.md

Nhiệm vụ: Implement role `roles/os_prep` và playbook `playbooks/os_prep.yml` theo đặc tả trong PHASE-2.

Ràng buộc:
- Idempotent tuyệt đối.
- Mỗi sub-task (packages, users, sysctl, hugepages, watchdog, firewall, time_sync) là file riêng trong tasks/, gọi qua import_tasks từ main.yml.
- Defaults phải có comment giải thích từng biến.
- Handler `reload sysctl` đúng theo Ansible conventions.

Hoàn thành xong:
1. Show `roles/os_prep/tasks/main.yml`.
2. Show `roles/os_prep/defaults/main.yml`.
3. Output `ansible-lint roles/os_prep/`.
4. Liệt kê acceptance criteria nào tôi có thể test ngay trên 1 node, criteria nào cần SSH vào 3 node.
```

---

## Phase 3 — etcd cluster

```
Đọc các file:
@docs/CLAUDE.md
@docs/ARCHITECTURE.md
@docs/skills/project-conventions.md
@docs/skills/etcd-spec.md
@docs/phases/PHASE-3-etcd.md

Nhiệm vụ: Implement role `roles/etcd` và playbook `playbooks/etcd.yml` theo PHASE-3 + etcd-spec.

Ràng buộc đặc biệt:
- Bootstrap pattern phải đúng: 3 node start gần đồng thời với initial-cluster-state=new.
- Handler `restart etcd` được thiết kế để playbook gọi phải dùng serial:1 (không restart đồng thời).
- Detect "đã bootstrap" qua sự tồn tại của /u01/etcd/member/ để skip task bootstrap-only khi chạy lại.
- Health verify dùng run_once + delegate_to + retry.

Hoàn thành xong:
1. Show template `etcd.conf.yml.j2`.
2. Show `playbooks/etcd.yml`.
3. Output dry-run `ansible-playbook playbooks/etcd.yml --check --diff`.
```

---

## Phase 4a — PostgreSQL + Patroni (SKELETON)

```
Đọc các file:
@docs/CLAUDE.md
@docs/ARCHITECTURE.md
@docs/skills/project-conventions.md
@docs/skills/patroni-spec.md
@docs/phases/PHASE-4-patroni.md

Nhiệm vụ: Implement PHẦN 4a (skeleton) của PHASE-4: tạo cấu trúc 2 role `postgresql` và `patroni` với tasks file rỗng/stub, templates đã có placeholder, NHƯNG chưa điền nội dung đầy đủ.

Yêu cầu:
- Cấu trúc tasks chia file theo phase (install / configure / bootstrap_leader / join_replica) — mỗi file là một include point Ansible có thể gọi qua import_role tasks_from.
- Template patroni.yml.j2 có MỌI placeholder cần (tham chiếu biến), nhưng có thể chứa `{# TODO 4b: điền tham số chi tiết #}` ở phần cụ thể.
- Systemd unit đầy đủ ngay từ 4a — phần này không cần phân giai đoạn.
- Mask postgresql.service khỏi systemd PGDG package.

KHÔNG đụng tới orchestration playbook `database.yml` — để cho 4b.

Hoàn thành:
1. Tree của 2 role.
2. Show systemd unit `patroni.service.j2`.
3. Show patroni.yml.j2 với phần placeholder rõ ràng.
```

---

## Phase 4b — PostgreSQL + Patroni (LOGIC)

```
Đọc các file:
@docs/CLAUDE.md
@docs/ARCHITECTURE.md
@docs/skills/project-conventions.md
@docs/skills/patroni-spec.md
@docs/phases/PHASE-4-patroni.md

Nhiệm vụ: Implement PHẦN 4b của PHASE-4:
(a) Điền đầy đủ template `patroni.yml.j2` theo patroni-spec (bootstrap.dcs, pg_hba, users, postgresql, watchdog).
(b) Viết `playbooks/database.yml` với orchestration đúng: cài binary song song → bootstrap leader đầu tiên → replica join từng cái (serial:1) → tạo user/db dify trên primary.
(c) Tạo `tasks/find_primary.yml` reuse được trong roles và playbooks khác (query /cluster của Patroni).

Ràng buộc:
- Đợi /primary trả 200 trước khi cho replica join.
- Đợi /replica trả 200 cho mỗi replica trước khi sang replica tiếp theo.
- no_log: true cho mọi task chứa password.
- Tạo user dify, database dify chạy delegate trên primary (động qua find_primary), KHÔNG hardcode db_nodes[0].

Hoàn thành:
1. Show `playbooks/database.yml`.
2. Show `roles/patroni/templates/patroni.yml.j2` (đã đầy đủ).
3. Hỏi tôi password để put vào vault — KHÔNG sinh password mẫu, KHÔNG commit.
4. Hướng dẫn tôi chạy: lệnh nào để init vault, lệnh nào để apply lần đầu.
```

---

## Phase 5 — K8s Frontend (HAProxy + PgBouncer)

```
Đọc các file:
@docs/CLAUDE.md
@docs/ARCHITECTURE.md
@docs/skills/project-conventions.md
@docs/skills/haproxy-spec.md
@docs/skills/pgbouncer-spec.md
@docs/skills/k8s-manifests-spec.md
@docs/phases/PHASE-5-k8s-frontend.md

Nhiệm vụ: Implement 2 role `haproxy` + `pgbouncer` và playbook `playbooks/k8s_frontend.yml`.

Ràng buộc đặc biệt:
- Module DUY NHẤT để tương tác K8s: kubernetes.core.k8s và kubernetes.core.k8s_info. CẤM dùng command: kubectl.
- Playbook chạy trên localhost (Ansible controller có kubeconfig).
- Task fetch SCRAM verifier dùng community.postgresql.postgresql_query, delegate_to: localhost, login_host = IP của primary động (qua find_primary từ phase 4b).
- Mọi task chứa SCRAM string hoặc password: no_log: true.
- Annotation checksum/config trên Deployment để rolling khi cfg đổi.
- Wait rollout xong mới kết thúc playbook.

Tái sử dụng `tasks/find_primary.yml` từ phase 4b.

Hoàn thành:
1. Show `playbooks/k8s_frontend.yml`.
2. Show 2 template chính: haproxy.cfg.j2 và pgbouncer.ini.j2.
3. Show task `fetch_verifiers.yml` (đã masked secret trong output).
4. Output dry-run.
```

---

## Phase 6 — Verification + Day-2

```
Đọc các file:
@docs/CLAUDE.md
@docs/ARCHITECTURE.md
@docs/skills/project-conventions.md
@docs/skills/patroni-spec.md
@docs/skills/etcd-spec.md
@docs/skills/haproxy-spec.md
@docs/skills/pgbouncer-spec.md
@docs/phases/PHASE-6-verification.md

Nhiệm vụ: Implement 4 playbook day-2:
- playbooks/verify.yml
- playbooks/switchover.yml
- playbooks/reinit_replica.yml
- playbooks/show_status.yml

Ràng buộc:
- verify.yml: ASSERT mạnh mẽ ở mỗi tầng, fail rõ ràng nếu sai. Không changed bao giờ.
- switchover.yml và reinit_replica.yml: BẮT BUỘC vars_prompt xác nhận trước khi tác động. Sai bước = downtime.
- show_status.yml: read-only, không changed.
- Tái sử dụng tasks/find_primary.yml.
- In output show_status.yml dạng bảng dễ đọc (dùng ansible.builtin.debug với msg | to_nice_yaml hoặc tự format).

Hoàn thành:
1. Show 4 playbook.
2. Demo output mong đợi của show_status.yml.
3. Liệt kê 3 kịch bản failure cố tình tôi có thể test verify.yml (vd: stop 1 etcd, kill 1 PG…).
```

---

## Mẫu prompt sửa lỗi giữa chừng

Nếu một phase apply ra lỗi, dùng prompt sửa nhanh:

```
Đọc các file:
@docs/CLAUDE.md
@docs/skills/project-conventions.md
@docs/phases/PHASE-N-<...>.md   <-- phase đang lỗi
@<file Ansible đang gây lỗi>

Lỗi tôi gặp:
<dán full traceback / output của ansible-playbook>

Hãy:
1. Chỉ ra root cause cụ thể (không đoán).
2. Đề xuất sửa TỐI THIỂU, KHÔNG refactor diện rộng.
3. Giải thích vì sao convention/spec đã đặt nên áp dụng cách này.
4. Sau khi sửa, chỉ ra acceptance criteria nào của phase cần test lại.
```

---

## Checklist cuối tour

Sau khi pass cả 6 phase:

- [ ] `ansible-lint` clean toàn repo.
- [ ] `ansible-playbook --check playbooks/site.yml` không changed.
- [ ] `playbooks/verify.yml` pass.
- [ ] Switchover thử tay được, cluster về stable.
- [ ] Dify (staging) đổi `DB_HOST` sang PgBouncer Service, traffic chạy bình thường.
- [ ] Vault chưa commit plaintext, `.vault_pass` chưa commit.
- [ ] README repo root cập nhật cách chạy.

Khi cả 7 mục tick, cluster sẵn sàng cho cutover production (xem section 13 trong tài liệu kiến trúc đầy đủ).
