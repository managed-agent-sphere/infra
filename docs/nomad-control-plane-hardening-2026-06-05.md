---
title: Nomad 控制面与沙箱节点加固记录
date: 2026-06-05
tags:
  - nomad
  - consul
  - agentsphere
  - sandbox
  - firecracker
  - dns
  - systemd
status: done
---

# Nomad 控制面与沙箱节点加固记录

## 摘要

本次巡检覆盖 Nomad/Consul 控制面、业务服务、沙箱节点和磁盘水位。集群已恢复为 3 server raft，所有 Nomad client 为 `ready`。主要处置集中在三类问题：控制面节点混跑业务服务、DNS 路由穿透、沙箱节点磁盘与负载不均衡。

已完成：

- `server-242`：修复 `api.iss-ruidong.com` 内网解析路由，移除公网 fallback。
- `server-15`：恢复 Nomad server，给 `codesphere.service` 增加 cgroup 内存保护，添加 4G swap。
- `server-242`：给 `codesphere.service` 增加同等 cgroup 内存保护，修正脱离 systemd 的孤儿进程，添加 4G swap。
- `sandbox-14`：恢复 `client-orchestrator` alloc。
- `sandbox-14` / `sandbox-243`：为 `/var/log/firecracker-*.log` 配置 3 小时轮转。
- `sandbox-243`：确认根盘压力主要来自 `/orchestrator/build`，不应直接删除活跃文件。

## 拓扑

| 节点 | IP | 角色 | 备注 |
|---|---:|---|---|
| `server-93` | `10.142.10.93` | Nomad server leader / Consul | 控制面 |
| `server-242` | `10.142.40.242` | Nomad server follower / Consul | 同时运行 `codesphere` |
| `server-15` | `10.142.70.15` | Nomad server follower / Consul | 同时运行 `codesphere` / `api` / `client-proxy` |
| `sandbox-14` | `10.142.70.14` | Nomad client | Firecracker sandbox 节点 |
| `sandbox-243` | `10.142.40.243` | Nomad client | Firecracker sandbox 节点，当前负载偏高 |

Nomad/Consul 本身很轻量；风险来自控制面节点上混跑的业务进程。如果业务进程没有 cgroup 限制，内存、CPU 或 IO 压力会直接影响 raft、memberlist 和 Nomad heartbeat。

## 当前状态

Nomad raft 已恢复为 3 节点：

```text
server-93.global   leader
server-242.global  follower
server-15.global   follower
```

Nomad client 状态：

```text
sandbox-14   ready
sandbox-243  ready
server-15    ready
server-242   ready
server-93    ready
```

关键 job：

```text
api               Running=2
client-proxy      Running=2
orchestrator      Running
template-manager  Running
```

## 处置记录

### DNS 路由

`server-242` 上 `api.iss-ruidong.com` 曾经通过阿里云 DNS `100.100.2.136/138` 解析到公网地址 `8.141.111.57`。内部 DNS `10.142.10.93` 的正确返回为 `10.142.40.246`。

根因是 systemd-resolved 中 `~iss-ruidong.com` routing domain 被注释，导致查询没有强制进入内部 DNS。

最终配置：

```ini
# /etc/systemd/resolved.conf.d/custom-dns.conf
[Resolve]
DNS=10.142.10.93
Domains=~iss-ruidong.com
```

```ini
# /etc/systemd/resolved.conf.d/consul.conf
[Resolve]
DNS=127.0.0.1:8600
DNSSEC=false
Domains=~consul
```

验证点：

```text
DNS Servers: 127.0.0.1:8600 10.142.10.93
DNS Domain: ~consul ~iss-ruidong.com
api.iss-ruidong.com -> 10.142.40.246
```

### server-15 恢复与保护

`server-15` 曾出现 Nomad memberlist/raft 失联，日志中可见持续内存压力：

```text
systemd-journald: Under memory pressure, flushing caches.
```

该节点内存约 30Gi，故障前无 swap；同时混跑 Nomad server、Consul、Docker alloc 和宿主机 `codesphere.service`。`api` / `client-proxy` 由 Nomad/Docker 限制，`codesphere.service` 原先没有 systemd 内存上限。

已添加：

```ini
# /etc/systemd/system/codesphere.service.d/10-resource-guard.conf
[Service]
MemoryHigh=20G
MemoryMax=24G
OOMPolicy=kill
OOMScoreAdjust=500
Restart=on-failure
RestartSec=5
```

已添加 4G swap：

```text
/swapfile none swap sw,pri=10 0 0
```

验证：

```text
MemoryHigh=21474836480
MemoryMax=25769803776
OOMPolicy=kill
OOMScoreAdjust=500
/swapfile file 4G 0B 10
```

### server-242 codesphere 保护

`server-242` 上曾出现 `codesphere.service` 为 `inactive`，但 8080 端口仍由 codesphere 进程监听。该进程位于 `/user.slice/.../session-*.scope`，不受 `codesphere.service` 的 cgroup 策略约束。

处理方式：

1. 添加与 `server-15` 相同的 `10-resource-guard.conf`。
2. 通过应用自身 stop 脚本停止旧孤儿进程。
3. 使用 `systemctl start codesphere` 重新拉起。
4. 添加 4G swap。

最终状态：

```text
ActiveState=active
SubState=running
ControlGroup=/system.slice/codesphere.service
MemoryHigh=21474836480
MemoryMax=25769803776
OOMPolicy=kill
OOMScoreAdjust=500
/swapfile file 4G 0B 10
```

健康检查：

```text
GET http://127.0.0.1:8080/health -> {"ok":true,"service":"Codesphere API"}
```

### sandbox-14 orchestrator

`sandbox-14` 上旧 `client-orchestrator` alloc 失败，直接 restart 返回 `alloc that should not run`。通过停止失败 alloc 触发重调度后恢复。

```bash
nomad alloc stop -detach 2964c249
```

恢复后：

```text
81a93437  sandbox-14  client-orchestrator  running
GET http://127.0.0.1:5008/health -> {"status":"healthy","version":"355921a"}
```

### 沙箱负载与磁盘

`sandbox-14` 曾长时间无法承载新 sandbox，恢复后存量 VM 不会自动迁移，因此 `sandbox-243` 明显更重。

| 节点 | Firecracker VM 数 | `/orchestrator/build` | `/var/log` | 状态 |
|---|---:|---:|---:|---|
| `sandbox-14` | 约 13 | 约 31G | 约 692M | 较轻 |
| `sandbox-243` | 约 30 | 约 249G | 约 12G | 偏重 |

`sandbox-243` 的根盘压力主要来自 `/orchestrator/build` 中的：

```text
*-memfile-*
*-rootfs.ext4-*
```

这些文件不是普通日志。部分 `memfile` / `rootfs` 正被 orchestrator 或 Firecracker 持有，不能按文件名或 mtime 直接删除。清理前必须区分活跃文件和孤儿文件。

另有单个 sandbox `ik9rqxw7kbpwz9uor5trx` CPU 约 `199%`，即约 2 个 CPU core。若非预期计算任务，应通过业务/orchestrator 正常停止 sandbox，避免直接杀 Firecracker 进程造成状态不一致。

### Firecracker 日志轮转

两个沙箱节点均已配置 Firecracker 日志轮转：

```ini
# /etc/logrotate.d/firecracker
/var/log/firecracker-*.log {
    hourly
    maxsize 256M
    rotate 7
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
    dateext
    dateformat -%Y%m%d-%s
    su root root
}
```

由 systemd timer 每 3 小时触发：

```ini
# /etc/systemd/system/firecracker-logrotate.timer
[Timer]
OnCalendar=*-*-* 00/3:00:00
AccuracySec=5m
Persistent=true
```

验证：`firecracker-logrotate.service` 在 `sandbox-14` 和 `sandbox-243` 上均执行成功。

## 标准检查命令

```bash
export NOMAD_ADDR=http://10.142.10.93:4646
nomad operator raft list-peers
nomad server members
nomad node status
nomad job status
```

```bash
systemctl status codesphere --no-pager -l
systemctl show codesphere \
  -p ActiveState \
  -p SubState \
  -p ControlGroup \
  -p MemoryHigh \
  -p MemoryMax \
  -p OOMPolicy \
  -p OOMScoreAdjust
swapon --show
```

```bash
pgrep -fc /fc-versions/.*/firecracker
du -sh /orchestrator/build /orchestrator/sandbox /var/log /var/lib/docker 2>/dev/null
find /orchestrator/build -xdev -type f -mmin -30 -printf '%s\n' \
  | awk '{s+=$1} END {printf "%.2fG\n", s/1024/1024/1024}'
```

## 后续工作

- 控制面隔离：Nomad server 节点尽量只运行 Nomad server 和 Consul；`api`、`client-proxy`、`codesphere` 放到 client 节点。
- 调度均衡：sandbox 调度应纳入 VM 数、磁盘水位、近期 build 增量、节点恢复时间和高 CPU sandbox 数。
- 安全 GC：为 `/orchestrator/build` 实现只读审计，再清理未被 `lsof` 持有且已结束的孤儿文件。
- 日志降级：评估将 Firecracker 从 `Debug` 降至 `Info` 或 `Warn`。
- raw_exec 收敛：避免 `no_cgroups = true` 的 raw_exec job 落到 Nomad server 节点。
