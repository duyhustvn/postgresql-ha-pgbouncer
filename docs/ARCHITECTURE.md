# ARCHITECTURE.md — Dify PG HA

Đặc tả kiến trúc đích cho Ansible deployment. Tài liệu nguồn dài hơn (đầy đủ giải thích/lý do) ở ngoài bundle — file này là phiên bản cô đọng làm reference cho Claude Code.

## Topology

```
[K8s cluster]
   Dify pods (api + worker)
       │
       ▼   Service: pgbouncer.db.svc.cluster.local:6432
   PgBouncer Deployment  (3 replica, transaction mode)
       │
       ▼   Service: pg-haproxy.db.svc.cluster.local:5000
   HAProxy   Deployment  (2 replica, httpchk → Patroni :8008/primary)
       │
       │  ──── ra ngoài K8s ────
       ▼
[3 PG nodes vật lý: 10.0.0.1, 10.0.0.2, 10.0.0.3]
   Mỗi node: Patroni + PostgreSQL 16 + etcd member
```

## Bảng thành phần

| Tầng | Số instance | Triển khai trên | Cổng |
|---|---|---|---|
| etcd | 3 (quorum) | Cùng 3 PG node (data dir riêng nếu được) | 2379/2380 |
| PostgreSQL 16 | 3 (1 primary + 2 replica) | 3 PG node | 5432 |
| Patroni | 1 trên mỗi PG node | 3 PG node | 8008 (REST) |
| HAProxy | 2 replica | K8s Deployment, namespace `db` | 5000 (frontend), 7000 (stats) |
| PgBouncer | 3 replica | K8s Deployment, namespace `db` | 6432 |

## Bảng cổng (đầy đủ)

| Cổng | Thuộc | Truy cập bởi |
|---|---|---|
| 2379 | etcd client | Patroni → etcd (cả 3 PG node) |
| 2380 | etcd peer | etcd ↔ etcd |
| 5432 | PostgreSQL | Patroni (local), HAProxy pod (qua mạng), replication |
| 8008 | Patroni REST | HAProxy (health-check), quản trị |
| 5000 | HAProxy frontend | PgBouncer pod |
| 6432 | PgBouncer | Dify pod (qua K8s Service) |
| 7000 | HAProxy stats | nội bộ debug |

## Luồng kết nối khi vận hành bình thường

```
Dify pod  ─►  ClusterIP svc pgbouncer.db.svc:6432
          ─►  PgBouncer pod (transaction pooling: gộp ~540 client → ≤50 backend conn / pod)
          ─►  ClusterIP svc pg-haproxy.db.svc:5000
          ─►  HAProxy pod (httpchk: chỉ pg primary mới UP)
          ─►  10.0.0.X:5432  (Patroni-managed PostgreSQL primary)
```

## Phép tính phễu (worst-case)

| Tầng | Số connection |
|---|---|
| Dify (3 pod api + 3 pod worker, SQLAlchemy pool đã bóp) | ~540 client conn |
| PgBouncer (3 replica × max_client_conn 2000) | nhận thoải mái |
| PgBouncer → HAProxy (3 replica × default_pool_size 50 + reserve 10) | ≤ 180 server conn |
| HAProxy → PG primary | passthrough, ≤ 180 |
| PG primary (max_connections = 200) | ≤ 180, dư 10% headroom |

Quy tắc kiểm soát: `pgbouncer_replicas × (default_pool_size + reserve_pool_size) ≤ pg_max_connections × 0.85`.

## Tham số nhạy cảm (đừng đổi mà không cân nhắc)

| Tham số | Giá trị | Lý do |
|---|---|---|
| `patroni.bootstrap.dcs.ttl` | 30s | Cân bằng giữa downtime failover và nhạy với glitch mạng |
| `pgbouncer.pool_mode` | transaction | Bắt buộc để gộp client; session mode sẽ không multiplex |
| `pgbouncer.server_lifetime` | 600s | Recycle nhanh sau failover |
| `pg.max_connections` | 200 | Dư cho PgBouncer; cao hơn sẽ kéo work_mem budget |
| `pg.work_mem` | 32MB | Per sort/hash NODE per query, không per-connection |
| `pg.wal_log_hints` | on | BẮT BUỘC cho `pg_rewind` |
| `patroni.watchdog.mode` | required (prod) / off (test) | Chống split-brain |

## Failover semantics

- TTL hết → leader race trong etcd → một replica được promote (Patroni gọi `pg_promote()`).
- HAProxy `httpchk` phát hiện primary mới qua `/primary`, mark UP.
- `on-marked-down shutdown-sessions` đóng session cũ → PgBouncer reconnect.
- Estimated downtime: **30-40s** với TTL 30s (rút được xuống ~15s nếu hạ TTL).
- Primary cũ sống lại → `pg_rewind` tự đồng bộ timeline, rejoin làm replica.

## Backup (đề cập, không trong scope Ansible chính)

- pgBackRest WAL archiving → object storage (S3/MinIO).
- Patroni bootstrap có thể restore từ pgBackRest. Sẽ làm trong giai đoạn day-2.

## Outside scope

Bundle Ansible này KHÔNG bao gồm:
- Setup K8s cluster (đã có sẵn)
- Provision 3 PG node OS (đã có sẵn, chỉ apply OS prep)
- Cài đặt monitoring stack (Prometheus/Grafana) — sẽ thêm sau
- Logical replication migration từ Pgpool cũ — sẽ làm thủ công khi cutover
