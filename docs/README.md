# Dify PostgreSQL HA — Claude Code Docs Bundle

Bộ tài liệu để Claude Code sinh ra một dự án Ansible triển khai cụm PostgreSQL HA cho Dify, dùng kiến trúc **Patroni + etcd + HAProxy + PgBouncer**.

## Cách dùng

1. **Khởi tạo repo Ansible trống** (Claude Code sẽ scaffold ở Phase 1):
   ```
   mkdir postgres-ha-ansible && cd postgres-ha-ansible
   git init
   ```
2. **Copy nguyên thư mục này** vào `docs/` của repo:
   ```
   cp -r <bundle>/* docs/
   ```
3. **Mở Claude Code** trong repo, mở `docs/PROMPTS.md`, paste từng prompt theo thứ tự phase.
4. Mỗi prompt đã chỉ rõ file context cần load — Claude Code không cần đoán.

## Triết lý

- Mỗi phase **đặc tả WHAT, không HOW**. Phép tả kiến trúc, contract, success criteria; Claude Code chọn cách hiện thực.
- Mỗi prompt **chỉ load file cần** (CLAUDE.md + ARCHITECTURE.md + 1-2 skill liên quan + 1 phase). Tránh nhồi cả repo vào context.
- **Scaffold trước, logic sau** với phase phức tạp (Phase 4 chia 4a/4b).

## Cấu trúc files

```
docs/
├── README.md                      <-- file này (cho người, không cho Claude Code)
├── CLAUDE.md                      <-- top context, Claude Code đọc mỗi lần
├── ARCHITECTURE.md                <-- spec kiến trúc cô đọng
├── PHASES.md                      <-- index 6 phase
├── PROMPTS.md                     <-- prompt sẵn để paste vào Claude Code
├── phases/
│   ├── PHASE-1-scaffold.md
│   ├── PHASE-2-os-prep.md
│   ├── PHASE-3-etcd.md
│   ├── PHASE-4-patroni.md
│   ├── PHASE-5-k8s-frontend.md
│   └── PHASE-6-verification.md
└── skills/
    ├── project-conventions.md     <-- Ansible style + inventory + secrets
    ├── etcd-spec.md
    ├── patroni-spec.md
    ├── haproxy-spec.md
    ├── pgbouncer-spec.md
    └── k8s-manifests-spec.md
```

## Điều kiện tiên quyết

- 3 PG node (Linux, đã SSH-able từ Ansible controller, sudo nopasswd).
- 1 K8s cluster với kubectl access và quyền tạo namespace `db`.
- Ansible controller có `ansible >= 2.16`, `python >= 3.10`, collection `kubernetes.core`, `community.postgresql`, `ansible.posix`.
- Mạng: K8s pod CIDR phải reach được 3 PG node trên cổng 5432, 8008.

## Thứ tự chạy

```
Phase 1 (scaffold)        →   Phase 2 (OS prep)
                              ↓
Phase 3 (etcd)            →   Phase 4 (Patroni + PG)
                              ↓
Phase 5 (K8s frontend)    →   Phase 6 (verification + day-2 playbooks)
```

Mỗi phase chạy xong, kiểm tra acceptance criteria trước khi sang phase tiếp.

## Lưu ý

Bundle này **không** chứa code Ansible thật — đó là việc của Claude Code. Bundle chỉ chứa spec để định hướng Claude Code sinh code đúng.
