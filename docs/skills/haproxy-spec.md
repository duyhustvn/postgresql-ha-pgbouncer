# skills/haproxy-spec.md

Spec cho role `haproxy` (chạy trong K8s). Phase liên quan: 5, 6.

## Vai trò

L4 TCP proxy duy nhất biết primary hiện tại trong cụm Patroni. Frontend bind 5000, backend là 3 PG node ở port 5432; chỉ node trả 200 trên `http://<node>:8008/primary` mới được route TCP.

## Triển khai

- **Vị trí**: K8s namespace `db`, Deployment 2 replica, Service ClusterIP.
- **Image**: `haproxy:2.9` (pin trong `group_vars/all/main.yml`).
- **Apply qua** `kubernetes.core.k8s` từ Ansible (KHÔNG kubectl raw).

## haproxy.cfg template

```
global
    maxconn 5000
    log stdout format raw local0

defaults
    mode tcp
    log global
    retries 2
    timeout connect 5s
    timeout client  60m
    timeout server  60m
    timeout check   5s

listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /
    stats refresh 5s

listen postgres-primary
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 2 rise 2 on-marked-down shutdown-sessions
{% for h in groups['db_nodes'] %}
    server {{ hostvars[h].patroni_name }} {{ hostvars[h].ansible_host }}:5432 maxconn 200 check port 8008
{% endfor %}
```

Giải thích các flag quan trọng:

| Flag | Tác dụng |
|---|---|
| `option httpchk GET /primary` | Health-check qua HTTP, không TCP |
| `http-check expect status 200` | Chỉ 200 mới healthy (Patroni trả 503 nếu không là primary) |
| `check port 8008` | Health-check vào cổng Patroni REST, không cổng PG |
| `inter 3s fall 2 rise 2` | Interval 3s, fail sau 2 lần fail, recover sau 2 lần OK |
| `on-marked-down shutdown-sessions` | Khi primary cũ bị mark down, đóng TCP session đang gắn vào để PgBouncer reconnect ngay |
| `maxconn 200` | Trần TCP session mỗi backend; khớp với PG `max_connections` |

## K8s resources cần sinh

| Kind | Tên | Mô tả |
|---|---|---|
| Namespace | `db` | shared với pgbouncer |
| ConfigMap | `pg-haproxy-config` | chứa `haproxy.cfg` đã render |
| Deployment | `pg-haproxy` | 2 replica, image `haproxy:2.9` |
| Service | `pg-haproxy` | ClusterIP, port 5000 → targetPort 5000 |
| NetworkPolicy | `pg-haproxy-allow` (optional) | chỉ pod label `app=pgbouncer` được gọi vào |

## Deployment chi tiết

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pg-haproxy
  namespace: db
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 1, maxSurge: 1 }
  selector: { matchLabels: { app: pg-haproxy } }
  template:
    metadata:
      labels: { app: pg-haproxy }
      annotations:
        # buộc rolling khi config đổi
        checksum/config: "{{ haproxy_cfg_checksum }}"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector: { matchLabels: { app: pg-haproxy } }
                topologyKey: kubernetes.io/hostname
      containers:
        - name: haproxy
          image: "{{ haproxy_image }}"
          ports:
            - { containerPort: 5000, name: postgres }
            - { containerPort: 7000, name: stats }
          volumeMounts:
            - { name: config, mountPath: /usr/local/etc/haproxy }
          livenessProbe:
            tcpSocket: { port: 5000 }
            periodSeconds: 10
          readinessProbe:
            tcpSocket: { port: 5000 }
            periodSeconds: 5
          resources:
            requests: { cpu: 100m, memory: 64Mi }
            limits:   { cpu: 500m, memory: 256Mi }
      volumes:
        - name: config
          configMap: { name: pg-haproxy-config }
```

Lưu ý:
- `checksum/config` annotation: dùng `sha256sum` của haproxy.cfg đã render → khi cfg đổi, Deployment rolling restart tự động.
- `podAntiAffinity` để 2 pod không cùng node.

## Service

```yaml
apiVersion: v1
kind: Service
metadata: { name: pg-haproxy, namespace: db }
spec:
  selector: { app: pg-haproxy }
  ports:
    - { name: postgres, port: 5000, targetPort: 5000 }
```

Internal ClusterIP — KHÔNG expose external.

## Quan sát

- Stats: port-forward 7000 hoặc tạo Ingress riêng (ngoài scope).
- Log: `kubectl logs -n db -l app=pg-haproxy -f`.

## Acceptance criteria

- 2 pod `pg-haproxy` `Running` và `Ready`.
- `kubectl exec -n db pod/pg-haproxy-xxx -- curl -s localhost:7000/` ra HTML stats.
- Từ pod tạm: `kubectl run psql --rm -it --image=postgres:16 -- psql -h pg-haproxy.db.svc -p 5000 -U dify -d dify -c 'SELECT inet_server_addr()'` trả về IP của PG primary.
- Switchover Patroni → trong vòng 10s, query tiếp theo qua HAProxy hit primary mới.
