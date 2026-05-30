# skills/k8s-manifests-spec.md

Quy ước chung khi sinh K8s manifest từ Ansible. Phase liên quan: 5.

## Module bắt buộc dùng

`kubernetes.core.k8s` — KHÔNG dùng `command: kubectl ...`.

```yaml
- name: Apply ConfigMap
  kubernetes.core.k8s:
    state: present
    namespace: "{{ k8s_namespace }}"
    definition: "{{ lookup('template', 'configmap.yaml.j2') | from_yaml }}"
```

## Namespace

Sinh 1 lần, idempotent:

```yaml
- name: Ensure namespace
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata: { name: "{{ k8s_namespace }}" }
```

## Labels & annotations chuẩn

Mọi resource dán label tối thiểu:

```yaml
metadata:
  labels:
    app: <pg-haproxy|pgbouncer>
    cluster: "{{ cluster_name }}"
    managed-by: ansible
```

Deployment có annotation `checksum/config` để rolling restart khi config đổi:

```yaml
template:
  metadata:
    annotations:
      checksum/config: "{{ <component>_cfg_checksum }}"
```

Tính checksum:
```yaml
- name: Compute haproxy config checksum
  ansible.builtin.set_fact:
    haproxy_cfg_checksum: "{{ (lookup('template', 'haproxy.cfg.j2')) | hash('sha256') }}"
```

## Service ClusterIP

Cả 2 Service đều internal:

```yaml
spec:
  type: ClusterIP
  selector: { app: <name> }
  ports: [{ port: <p>, targetPort: <p> }]
```

KHÔNG dùng `type: LoadBalancer` hay `NodePort` cho tầng DB (security).

## NetworkPolicy (tùy chọn nhưng khuyến nghị)

Hạn chế ai gọi được vào tầng DB:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pgbouncer-allow-dify
  namespace: db
spec:
  podSelector: { matchLabels: { app: pgbouncer } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { name: dify } }
      ports: [{ protocol: TCP, port: 6432 }]
```

Tương tự cho HAProxy (chỉ pgbouncer namespace=db gọi vào).

CNI phải hỗ trợ NetworkPolicy (Calico, Cilium). Nếu CNI không hỗ trợ (Flannel default), skip task này nhưng vẫn render manifest để document.

## Secret handling

- Tạo qua `kubernetes.core.k8s` với `stringData` (không cần base64 thủ công).
- `no_log: true` cho task chứa secret content.
- KHÔNG commit YAML có giá trị thật của Secret.

```yaml
- name: Create pgbouncer userlist secret
  kubernetes.core.k8s:
    state: present
    namespace: "{{ k8s_namespace }}"
    definition:
      apiVersion: v1
      kind: Secret
      metadata: { name: pgbouncer-userlist }
      type: Opaque
      stringData:
        userlist.txt: "{{ rendered_userlist_content }}"
  no_log: true
```

## ConfigMap

```yaml
- name: Create pgbouncer config
  kubernetes.core.k8s:
    state: present
    namespace: "{{ k8s_namespace }}"
    definition:
      apiVersion: v1
      kind: ConfigMap
      metadata: { name: pgbouncer-config }
      data:
        pgbouncer.ini: "{{ lookup('template', 'pgbouncer.ini.j2') }}"
```

## PodDisruptionBudget (khuyến nghị)

Để rolling node K8s không kill quá nhiều pod:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: pgbouncer, namespace: db }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: pgbouncer } }
```

Tương tự cho HAProxy với `minAvailable: 1`.

## Resource requests/limits

Bắt buộc set cho mọi container (để K8s scheduler đặt pod đúng):

| Component | CPU req | CPU lim | Mem req | Mem lim |
|---|---|---|---|---|
| HAProxy | 100m | 500m | 64Mi | 256Mi |
| PgBouncer | 100m | 1000m | 128Mi | 512Mi |

PgBouncer single-thread, lim CPU cao hơn để không bị throttle dưới tải.

## Probe

| Type | HAProxy | PgBouncer |
|---|---|---|
| Liveness | tcpSocket :5000, periodSeconds 10 | tcpSocket :6432, periodSeconds 10 |
| Readiness | tcpSocket :5000, periodSeconds 5 | tcpSocket :6432, periodSeconds 5 |

Lưu ý: TCP probe chỉ check process alive — PgBouncer có thể up nhưng PG backend down. Nâng cao: dùng `exec` probe chạy `psql -h /tmp -p 6432 -U pgbouncer_admin pgbouncer -c "SHOW POOLS"` (cần auth, phức tạp hơn — phase day-2).

## Apply order

```
1. Namespace
2. ConfigMap (haproxy + pgbouncer)
3. Secret  (pgbouncer-userlist) 
4. Deployment (haproxy trước, vì pgbouncer phụ thuộc Service haproxy)
5. Service (haproxy + pgbouncer)
6. NetworkPolicy + PDB
```

Ansible `kubernetes.core.k8s` xử lý tự động dependency qua selector — nhưng ordering rõ ràng giúp debug dễ hơn.

## kubeconfig

Ansible controller cần `~/.kube/config` hoặc env `KUBECONFIG` trỏ tới cluster đích. Module `kubernetes.core.k8s` đọc tự động. Nếu dùng service account in-cluster, set `KUBECONFIG` rỗng và để module tự detect.

## Validate sau apply

```yaml
- name: Wait for haproxy rollout
  kubernetes.core.k8s_info:
    api_version: apps/v1
    kind: Deployment
    name: pg-haproxy
    namespace: "{{ k8s_namespace }}"
  register: haproxy_dep
  until: >
    haproxy_dep.resources | length > 0 and
    haproxy_dep.resources[0].status.readyReplicas | default(0) ==
    haproxy_dep.resources[0].spec.replicas
  retries: 30
  delay: 5
```

Áp dụng tương tự cho pgbouncer.
