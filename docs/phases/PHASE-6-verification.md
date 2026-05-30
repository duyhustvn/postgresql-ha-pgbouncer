# PHASE-6 — Verification & Day-2 Playbooks

## Mục tiêu

Sinh playbook để (a) verify cluster sau cài đặt, (b) thực hiện switchover có kế hoạch, (c) reinit replica lag/hỏng. Đây là toolbox vận hành tối thiểu trước khi đưa cluster vào production.

## In scope

- `playbooks/verify.yml`: smoke test xuyên 4 tầng (etcd, Patroni, HAProxy, PgBouncer).
- `playbooks/switchover.yml`: thực hiện `patronictl switchover` với xác nhận, đợi state stable.
- `playbooks/reinit_replica.yml`: chọn 1 replica để reinit (basebackup lại), kiểm sau khi xong nó streaming OK.
- `playbooks/show_status.yml`: in trạng thái gọn (cluster topo, replication lag, PgBouncer pools, HAProxy backend health).

## Out of scope

- Backup/restore với pgBackRest — day-2 tách phase riêng.
- Monitoring stack (Prometheus, Grafana, alerts).
- Logical replication migration từ Pgpool cũ.

## Deliverables

```
playbooks/verify.yml
playbooks/switchover.yml
playbooks/reinit_replica.yml
playbooks/show_status.yml
```

Có thể sinh thêm `roles/verify/` nếu task verify dài, nhưng playbook flat là đủ.

## Context files cần load

- `docs/CLAUDE.md`
- `docs/ARCHITECTURE.md`
- `docs/skills/project-conventions.md`
- `docs/skills/patroni-spec.md`
- `docs/skills/etcd-spec.md`
- `docs/skills/haproxy-spec.md`
- `docs/skills/pgbouncer-spec.md`
- `docs/phases/PHASE-6-verification.md` (file này)

## Implementation notes

### verify.yml

Cấu trúc:
```yaml
- hosts: etcd
  run_once: true
  tasks:
    - name: etcd health
      ansible.builtin.command: >
        etcdctl --endpoints={{ etcd_endpoints }} endpoint health --cluster
      changed_when: false

- hosts: db_nodes
  run_once: true
  tasks:
    - name: Patroni cluster state
      ansible.builtin.command: >
        /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list -f json
      register: patroni_list
      changed_when: false
    - name: Assert 1 Leader + 2 Replica streaming
      ansible.builtin.assert:
        that:
          - (patroni_list.stdout | from_json | selectattr('Role','eq','Leader') | list | length) == 1
          - (patroni_list.stdout | from_json | selectattr('Role','eq','Replica') | selectattr('State','eq','streaming') | list | length) == 2

- hosts: localhost
  connection: local
  tasks:
    - name: HAProxy pods ready
      kubernetes.core.k8s_info:
        api_version: apps/v1
        kind: Deployment
        name: pg-haproxy
        namespace: db
      register: hp
    - assert: { that: hp.resources[0].status.readyReplicas == hp.resources[0].spec.replicas }

    - name: PgBouncer pods ready
      kubernetes.core.k8s_info: { ... pgbouncer ... }
      register: pb
    - assert: { that: pb.resources[0].status.readyReplicas == pb.resources[0].spec.replicas }

    - name: End-to-end SELECT 1 qua PgBouncer
      kubernetes.core.k8s_exec:
        namespace: db
        pod: "{{ pb.resources[0].metadata.name }}"
        command: psql 'postgresql://dify:{{ dify_db_password }}@pgbouncer.db.svc:6432/dify' -tAc 'SELECT 1'
      register: e2e
      no_log: true
    - assert: { that: "'1' in e2e.stdout" }
```

### switchover.yml

```yaml
- hosts: db_nodes
  vars_prompt:
    - name: target_leader
      prompt: "Switchover target node (pg1/pg2/pg3)?"
      private: false
    - name: confirm
      prompt: "Type CONFIRM to proceed"
      private: false
  pre_tasks:
    - fail: { msg: "Aborted" }
      when: confirm != "CONFIRM"
  tasks:
    - name: Find current leader
      # qua REST /cluster
      ...
    - name: Perform switchover
      ansible.builtin.command: >
        /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml
        switchover --leader {{ current_leader }} --candidate {{ target_leader }} --force
      run_once: true
      delegate_to: "{{ groups['db_nodes'][0] }}"
    - name: Wait for new leader stable
      ansible.builtin.uri:
        url: "http://{{ hostvars[target_leader].ansible_host }}:8008/primary"
        status_code: 200
      retries: 30
      delay: 2
```

### reinit_replica.yml

```yaml
- hosts: db_nodes
  vars_prompt:
    - name: replica_to_reinit
      prompt: "Replica to reinit (pg2/pg3)?"
  tasks:
    - name: Verify target is NOT current leader
      ...
    - name: Reinit
      ansible.builtin.command: >
        /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml
        reinit {{ cluster_name }} {{ replica_to_reinit }} --force
      run_once: true
      delegate_to: "{{ groups['db_nodes'][0] }}"
    - name: Wait for streaming
      ...
```

### show_status.yml

In bảng đẹp gồm:
- `patronictl list`
- HAProxy backend status (curl stats)
- PgBouncer `SHOW POOLS`
- Replication lag từ `pg_stat_replication`

Không assert, không thay đổi gì — read-only.

## Acceptance criteria

1. `ansible-playbook playbooks/verify.yml` chạy sạch (mọi assert pass) khi cluster healthy.
2. `verify.yml` fail rõ ràng khi cố tình stop 1 etcd hoặc 1 PG replica.
3. `switchover.yml` thực hiện thành công, cluster ổn định sau 60s, PgBouncer query bình thường.
4. `reinit_replica.yml` reinit replica đã chọn, sau đó replica streaming lag 0.
5. `show_status.yml` in output sạch, dễ đọc.
6. Tất cả playbook idempotent ở task không phải trigger action (verify, show_status không changed bao giờ).
7. `ansible-lint` clean.
