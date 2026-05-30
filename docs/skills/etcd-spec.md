# skills/etcd-spec.md

Spec cho role `etcd`. Phase liên quan: 3, 6.

## Phiên bản & nguồn

- etcd `v3.5.13` (pin trong `group_vars/all/main.yml` qua biến `etcd_version`).
- Tải từ GitHub release: `https://github.com/etcd-io/etcd/releases/download/{{ etcd_version }}/etcd-{{ etcd_version }}-linux-amd64.tar.gz`.
- Binary đặt vào `/usr/local/bin/etcd` và `/usr/local/bin/etcdctl`, mode 0755, owner root.

## User & directories

- User hệ thống: `etcd:etcd`, no login shell.
- Data dir: `/var/lib/etcd`, mode 0700, owner `etcd:etcd`.
- Config dir: `/etc/etcd`, mode 0750.
- Config file: `/etc/etcd/etcd.conf.yml`.

## Config template (etcd.conf.yml.j2)

Các tham số bắt buộc (giá trị lấy từ inventory):

```yaml
name: '{{ etcd_name }}'                       # từ hostvars (etcd1/etcd2/etcd3)
data-dir: /var/lib/etcd
listen-client-urls: 'http://{{ ansible_host }}:2379,http://127.0.0.1:2379'
advertise-client-urls: 'http://{{ ansible_host }}:2379'
listen-peer-urls: 'http://{{ ansible_host }}:2380'
initial-advertise-peer-urls: 'http://{{ ansible_host }}:2380'
initial-cluster: '{% for h in groups["etcd"] %}{{ hostvars[h].etcd_name }}=http://{{ hostvars[h].ansible_host }}:2380{% if not loop.last %},{% endif %}{% endfor %}'
initial-cluster-token: '{{ cluster_name }}-etcd'
initial-cluster-state: 'new'
auto-compaction-retention: '1'
heartbeat-interval: 100
election-timeout: 1000
```

> Lưu ý: `initial-cluster-state: 'new'` chỉ đúng khi BOOTSTRAP. Nếu add node mới sau này phải set `existing` — đây là edge case ngoài scope phase 3.

## Systemd unit (etcd.service)

```
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd
After=network-online.target
Wants=network-online.target

[Service]
User=etcd
Group=etcd
Type=notify
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf.yml
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## Thứ tự khởi động bootstrap

Trong phase 3, cluster CHƯA tồn tại nên cả 3 node phải start gần đồng thời. Pattern:

1. Template config + systemd unit trên cả 3 node (`serial` không cần ở đây — parallel OK).
2. `systemctl daemon-reload` trên cả 3.
3. `systemctl enable --now etcd` trên cả 3 (trong khoảng 1 task — Ansible tự song song).
4. Wait `etcd_health_check_timeout` (60s) rồi verify.

## Health check task

```yaml
- name: Wait for etcd cluster healthy
  ansible.builtin.command:
    cmd: >
      etcdctl --endpoints={{ groups['etcd'] | map('extract', hostvars, 'ansible_host')
                          | map('regex_replace', '^', 'http://')
                          | map('regex_replace', '$', ':2379') | join(',') }}
      endpoint health --cluster
  register: etcd_health
  changed_when: false
  retries: 12
  delay: 5
  until: etcd_health.rc == 0
  run_once: true
  delegate_to: "{{ groups['etcd'] | first }}"
```

## Quy tắc vận hành

- **Không bao giờ chạy với 2 node** (mất quorum dễ).
- Snapshot backup hằng ngày: `etcdctl snapshot save /var/backups/etcd/snapshot-$(date +%F).db` qua cronjob (sẽ làm ở phase 6 day-2).
- Monitor `etcd_disk_wal_fsync_duration_seconds` (p99 < 50ms).
- Khi cần restart 1 node, làm tuần tự (1 node 1 lần) chờ healthy mới sang node tiếp theo. Role phải có handler `restart etcd` cấu hình `serial: 1` ở playbook gọi role.

## Acceptance criteria phase 3

- `etcdctl endpoint health` trả `healthy` cho cả 3 endpoint.
- `etcdctl endpoint status -w table` cho thấy 1 IS_LEADER=true.
- Chạy lại playbook không có task `changed`.
- `systemctl status etcd` trên cả 3 node = active (running).
