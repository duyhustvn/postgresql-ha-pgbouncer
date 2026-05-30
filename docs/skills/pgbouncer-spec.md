# skills/pgbouncer-spec.md

Spec cho role `pgbouncer` (chạy trong K8s). Phase liên quan: 5, 6.

## Vai trò

Transaction-mode connection pooler. Nhận client connection từ Dify, gộp lại, mở ít connection backend tới HAProxy → PG primary. Đây là **tầng QUYẾT ĐỊNH bảo vệ PG khỏi OOM**.

## Triển khai

- **Vị trí**: K8s namespace `db`, Deployment 3 replica, Service ClusterIP.
- **Image**: `edoburu/pgbouncer:1.23` (pin biến `pgbouncer_image`). Hỗ trợ Prometheus exporter sidecar nếu cần.
- **Apply qua** `kubernetes.core.k8s`.

## pgbouncer.ini template

```
[databases]
dify = host=pg-haproxy.db.svc.cluster.local port=5000 dbname=dify

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
admin_users = pgbouncer_admin
stats_users = pgbouncer_admin

pool_mode = transaction
server_reset_query = DISCARD ALL
server_reset_query_always = 0

max_client_conn = 2000
default_pool_size = 50
min_pool_size = 10
reserve_pool_size = 10
reserve_pool_timeout = 3
max_db_connections = 100

server_lifetime = 600
server_idle_timeout = 120
server_connect_timeout = 15
query_wait_timeout = 120

ignore_startup_parameters = extra_float_digits,application_name

logfile = /dev/stdout
log_connections = 0
log_disconnections = 0
```

## userlist.txt — SCRAM verifier

PgBouncer cần biết SCRAM verifier của user `dify`. Verifier lấy động từ PG sau khi user được tạo (phase 4).

Workflow:
1. Phase 4 tạo user `dify` với password.
2. Phase 5 query PG primary: `SELECT rolname, rolpassword FROM pg_authid WHERE rolname IN ('dify', 'pgbouncer_admin');`
3. Render verifier vào template userlist.txt:
   ```
   "dify" "SCRAM-SHA-256$4096:...salt...$...storedkey...:...serverkey..."
   "pgbouncer_admin" "SCRAM-SHA-256$..."
   ```
4. Tạo K8s Secret từ file đó.

Ansible:
```yaml
- name: Fetch SCRAM verifiers
  community.postgresql.postgresql_query:
    db: postgres
    login_user: postgres
    login_host: "{{ hostvars[groups['db_nodes'][0]].ansible_host }}"
    login_password: "{{ postgres_superuser_password }}"
    query: "SELECT rolname, rolpassword FROM pg_authid WHERE rolname = ANY(%s)"
    positional_args:
      - ['dify', 'pgbouncer_admin']
  register: scram_verifiers
  no_log: true
  delegate_to: localhost

- name: Render userlist.txt
  ansible.builtin.template:
    src: userlist.txt.j2
    dest: "{{ playbook_dir }}/.cache/userlist.txt"
    mode: '0600'
  vars:
    verifiers: "{{ scram_verifiers.query_result }}"
  no_log: true

- name: Create K8s Secret pgbouncer-userlist
  kubernetes.core.k8s:
    state: present
    namespace: "{{ k8s_namespace }}"
    definition:
      apiVersion: v1
      kind: Secret
      metadata: { name: pgbouncer-userlist }
      type: Opaque
      stringData:
        userlist.txt: "{{ lookup('file', playbook_dir + '/.cache/userlist.txt') }}"
  no_log: true
```

`.cache/userlist.txt` thêm vào `.gitignore`.

## K8s resources cần sinh

| Kind | Tên | Mô tả |
|---|---|---|
| ConfigMap | `pgbouncer-config` | chứa `pgbouncer.ini` |
| Secret | `pgbouncer-userlist` | chứa `userlist.txt` với SCRAM verifier |
| Deployment | `pgbouncer` | 3 replica |
| Service | `pgbouncer` | ClusterIP, port 6432 |
| NetworkPolicy | (optional) | chỉ Dify namespace gọi vào |

## Deployment chi tiết

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: db
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 1, maxSurge: 1 }
  selector: { matchLabels: { app: pgbouncer } }
  template:
    metadata:
      labels: { app: pgbouncer }
      annotations:
        checksum/config: "{{ pgbouncer_ini_checksum }}"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector: { matchLabels: { app: pgbouncer } }
                topologyKey: kubernetes.io/hostname
      containers:
        - name: pgbouncer
          image: "{{ pgbouncer_image }}"
          env:
            - name: DB_USER
              value: "dify"
          ports:
            - { containerPort: 6432, name: pgbouncer }
          volumeMounts:
            - { name: config-ini, mountPath: /etc/pgbouncer/pgbouncer.ini, subPath: pgbouncer.ini }
            - { name: userlist,    mountPath: /etc/pgbouncer/userlist.txt,  subPath: userlist.txt }
          livenessProbe:
            tcpSocket: { port: 6432 }
            periodSeconds: 10
          readinessProbe:
            tcpSocket: { port: 6432 }
            periodSeconds: 5
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 1000m, memory: 512Mi }
      volumes:
        - name: config-ini
          configMap: { name: pgbouncer-config }
        - name: userlist
          secret:
            secretName: pgbouncer-userlist
            defaultMode: 0400
```

## Service

```yaml
apiVersion: v1
kind: Service
metadata: { name: pgbouncer, namespace: db }
spec:
  selector: { app: pgbouncer }
  ports: [{ port: 6432, targetPort: 6432 }]
```

Dify connect: `DB_HOST=pgbouncer.db.svc.cluster.local`, `DB_PORT=6432`.

## Acceptance criteria

- 3 pod `pgbouncer` Running + Ready.
- Từ pod tạm trong K8s:
  ```
  kubectl run psql --rm -it --image=postgres:16 -- \
    psql -h pgbouncer.db.svc -p 6432 -U dify -d dify -c "SELECT 1"
  ```
  trả 1.
- `kubectl exec -n db pgbouncer-xxx -- psql -h /tmp -p 6432 -U pgbouncer_admin pgbouncer -c "SHOW POOLS"` cho thấy `cl_active`, `sv_active` đếm được.
- Tải đồng thời 100 client (qua pgbench hoặc tool tương đương) → `SHOW POOLS` thấy `sv_active ≤ 50` (default_pool_size), `cl_waiting` thấp/0.
- Sau Patroni switchover, query mới qua PgBouncer thành công trong < 60s (do `server_lifetime` + retry).

## Lưu ý vận hành

- KHÔNG bật `application_name_add_host = 1` cho transaction mode trừ khi đã hiểu kỹ — có thể vô hiệu hóa pooling.
- Khi đổi `pool_mode` hoặc `default_pool_size`, phải rolling restart Deployment (không reload được trong PgBouncer).
- Sidecar Prometheus exporter (`prometheuscommunity/pgbouncer-exporter`) sẽ thêm ở phase day-2 monitoring.
