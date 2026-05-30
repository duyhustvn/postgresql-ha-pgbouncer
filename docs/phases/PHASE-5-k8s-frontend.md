# PHASE-5 — K8s Frontend Tier (HAProxy + PgBouncer)

## Mục tiêu

Triển khai 2 Deployment trong K8s (namespace `db`): HAProxy (route tới Patroni primary) + PgBouncer (transaction pooling). Dify sẽ connect vào `pgbouncer.db.svc.cluster.local:6432`.

## In scope

- Role `haproxy`: render `haproxy.cfg` từ inventory (3 PG node, port 8008 health-check, port 5432 backend), apply Namespace + ConfigMap + Deployment + Service.
- Role `pgbouncer`: render `pgbouncer.ini`, query SCRAM verifier từ PG primary, tạo Secret `userlist.txt`, apply Deployment + Service.
- Task `find_primary` reuse từ phase 4: tìm primary động qua REST `/cluster` để query verifier.
- (Tùy chọn) NetworkPolicy + PodDisruptionBudget.

## Out of scope

- Ingress, LoadBalancer, exposure ra ngoài cluster — DB tier internal-only.
- Prometheus exporter sidecar — day-2.
- HPA cho PgBouncer — day-2 sau khi đo tải thực.

## Deliverables

```
roles/haproxy/
├── tasks/
│   ├── main.yml
│   ├── render.yml           # tính checksum cfg, set_fact
│   └── apply.yml            # ns, cm, deploy, svc, (npol, pdb)
├── templates/
│   ├── haproxy.cfg.j2
│   ├── configmap.yaml.j2
│   ├── deployment.yaml.j2
│   ├── service.yaml.j2
│   └── networkpolicy.yaml.j2
├── defaults/main.yml
├── meta/main.yml
└── README.md

roles/pgbouncer/
├── tasks/
│   ├── main.yml
│   ├── fetch_verifiers.yml  # query SCRAM verifier từ PG primary
│   ├── render.yml
│   └── apply.yml
├── templates/
│   ├── pgbouncer.ini.j2
│   ├── userlist.txt.j2
│   ├── configmap.yaml.j2
│   ├── secret.yaml.j2
│   ├── deployment.yaml.j2
│   └── service.yaml.j2
├── defaults/main.yml
├── meta/main.yml
└── README.md
```

```
playbooks/k8s_frontend.yml
```

## Context files cần load

- `docs/CLAUDE.md`
- `docs/ARCHITECTURE.md`
- `docs/skills/project-conventions.md`
- `docs/skills/haproxy-spec.md`
- `docs/skills/pgbouncer-spec.md`
- `docs/skills/k8s-manifests-spec.md`
- `docs/phases/PHASE-5-k8s-frontend.md` (file này)

## Implementation notes

- Playbook chạy với `hosts: localhost`, `connection: local` (Ansible controller có kubeconfig).
- Một task ban đầu phải resolve primary động:
  ```yaml
  - name: Find current Patroni primary
    ansible.builtin.uri:
      url: "http://{{ hostvars[item].ansible_host }}:8008/primary"
      status_code: 200
    register: pri_check
    failed_when: false
    changed_when: false
    delegate_to: localhost
    loop: "{{ groups['db_nodes'] }}"
  
  - ansible.builtin.set_fact:
      pg_primary_host: "{{ (pri_check.results | selectattr('status', 'eq', 200) | list | first).item }}"
  ```
- Fetch SCRAM verifier `delegate_to: localhost` nhưng `login_host` là `hostvars[pg_primary_host].ansible_host`. Yêu cầu PG bind IP đó cho dải Ansible controller (đã làm trong phase 4 pg_hba với `network_cidr`). Có thể cần SSH tunnel — note trong README role.
- Checksum config để Deployment rolling khi cfg đổi:
  ```yaml
  - ansible.builtin.set_fact:
      haproxy_cfg_rendered: "{{ lookup('template', 'haproxy.cfg.j2') }}"
      haproxy_cfg_checksum: "{{ (lookup('template', 'haproxy.cfg.j2')) | hash('sha256') }}"
  ```
  Annotation `checksum/config` lấy giá trị này.
- `kubernetes.core.k8s` với `definition: "{{ lookup('template', ...) | from_yaml }}"`. KHÔNG dùng `k8s_apply` (deprecated).
- Wait rollout (xem skill k8s-manifests-spec).
- `no_log: true` cho task tạo Secret và task fetch verifier (cả 2 chứa SCRAM string nhạy cảm).

## Acceptance criteria

1. `ansible-playbook playbooks/k8s_frontend.yml` lần đầu thành công.
2. `kubectl -n db get pods` thấy 2 pod `pg-haproxy-*` + 3 pod `pgbouncer-*`, tất cả `1/1 Running` + `Ready`.
3. `kubectl -n db get svc` thấy `pg-haproxy` và `pgbouncer` ClusterIP.
4. Test connection từ pod tạm:
   ```
   kubectl run pgtest --rm -it --image=postgres:16 --restart=Never -- \
     psql 'postgresql://dify:<pwd>@pgbouncer.db.svc.cluster.local:6432/dify' \
     -c 'SELECT inet_server_addr(), now()'
   ```
   Trả về IP của PG primary hiện tại.
5. `kubectl exec -n db <pgbouncer-pod> -- psql -h /tmp -p 6432 -U pgbouncer_admin pgbouncer -c "SHOW POOLS"` ra bảng pools.
6. Patroni switchover → trong < 60s, query mới qua PgBouncer thành công (có thể fail 1-2 lần đầu rồi recover qua retry/server_lifetime).
7. Apply lại playbook → KHÔNG changed (kể cả Secret — `kubernetes.core.k8s` so sánh field).
8. `ansible-lint` clean.

## Cảnh báo

- **Proxy & image pull**: Bundle Ansible này không provision K8s worker node. Nếu K8s worker node cần proxy để pull image từ Docker Hub (`haproxy:2.9`, `edoburu/pgbouncer:1.23`), phải cấu hình containerd proxy **trước** khi chạy phase này — nằm ngoài scope bundle. Tham khảo: tạo file `/etc/systemd/system/containerd.service.d/proxy.conf` trên mỗi K8s node:
  ```ini
  [Service]
  Environment="HTTP_PROXY=http://proxy.company.com:3128"
  Environment="HTTPS_PROXY=http://proxy.company.com:3128"
  Environment="NO_PROXY=localhost,127.0.0.1,10.244.0.0/16,.svc.cluster.local"
  ```
  Sau đó `systemctl daemon-reload && systemctl restart containerd`. Chỉ cần làm một lần khi setup cluster.
- Nếu NetworkPolicy bật mà CNI không hỗ trợ → traffic vẫn pass nhưng policy không hiệu lực. Document trong role README.
- `kubernetes.core.k8s` với Secret `stringData` lưu base64 trong etcd K8s (không phải plaintext etcd-side, nhưng ai có quyền GET Secret đều decode được). Không phải secret-at-rest đúng nghĩa — production cần SealedSecrets/Vault. Đề cập trong role README, để hậu kỳ.
