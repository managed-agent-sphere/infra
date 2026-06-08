# Nomad orchestrator 故障复盘 — 2026-06-08

## 一、症状

```
nomad alloc logs <id>   → 404, state for allocation not found
nomad job status        → orchestrator: 7 Failed, 0 Running (desired=run)
```

## 二、排查时间线

| 时刻 | 动作 | 发现 |
|---|---|---|
| — | `nomad node status` | 4 个 node 全 ready,排除节点宕机 |
| — | `nomad job eval orchestrator` | 新 alloc 立即 failed |
| — | `nomad alloc status 1ec922d3` | "Setup Failure: no space left on device" |
| — | `df -h` 误查 cntl1297 | 看错机器(失败在 codesphere) |
| — | 通过 Nomad API 查 codesphere | `bytesfree=773G/940G`,磁盘其实没满,旧报错已自愈 |
| — | `nomad alloc logs 1ec922d3` | **真正错误**:`FATAL Could not connect to Redis: dial tcp 10.96.8.223:5432: i/o timeout` |
| — | 探测 `10.96.8.223:5432` | CLOSED — 一开始假设是 k8s svc 挂了 |
| — | 用户提示 Redis 在 Nomad `api.hcl` 里 | 推翻 k8s 假设 |
| — | 对比 72/73 上的 `orchestrator.hcl` | **配置漂移**:72 是 `:5432`,73 是 `:6379` |
| — | 探测 `:6379` 6 个节点 | **全部 OPEN** — Redis 集群健康 |
| — | 看 deployed Job Version 26 SubmitTime | 今天 12:02:29 用了 72 上的错文件 redeploy |

## 三、根因

**配置漂移 + 部署源不规范**:

- 72 (cntl1297) 上的 `/opt/app-deployment/jobs/orchestrator.hcl` 端口写错成 `:5432`(PG 端口),自 Jan 21 起就是错的
- 73 (cntl1298) 上的同名文件在 6/1 被纠正为 `:6379`,但**没同步到 72**
- 74 (cntl1299) 上压根没这个目录
- 今天 12:02 有人在 72 上 `nomad job run` → 把错配置推上集群 → orchestrator 连不上 Redis → 反复 exit 1 → `Not Restarting`

附带的旧"磁盘满"报错(3 个早期 alloc)是次生现象,Nomad 自己 GC 后已恢复,**不是根因**。

### 关键证据

| 验证项 | 结果 |
|---|---|
| 部署的 Job Version 26 实际 env | `REDIS_CLUSTER_URL = ...10.96.8.223:5432...` |
| Redis 真实端口 6379 — 6 个节点全部 | 全部 OPEN(Redis 集群健康) |
| 部署用的 5432 端口 — 10.96.8.223 | CLOSED |
| 10.96.8.222:5432 (PG) | OPEN(印证 5432 = PG 端口) |

| 文件 | orchestrator.hcl 中的 Redis 端口 |
|---|---|
| 72 (cntl1297) 当前 | `:5432` ❌ |
| 73 (cntl1298) 当前 | `:6379` ✓ |
| 73 上 6/1 备份 | `:6379` ✓ |
| 73 上 1/27 备份 | `:6379` ✓ |
| 74 (cntl1299) | 没有 `/opt/app-deployment/jobs/` 目录 |

## 四、修复

**一行命令**(在 73 上,因为 73 的 hcl 是正确的):

```bash
nomad job run /opt/app-deployment/jobs/orchestrator.hcl
```

部署后 orchestrator 会用 `:6379` 连 Redis,自动恢复。template-manager(也长期没 alloc)很可能同样依赖 Redis,会一起恢复。

## 五、以后遇到类似问题的标准排查路径

### 阶段 1 — Nomad 层(看调度/启动是不是过了)

```bash
nomad node status                    # 节点是否健康
nomad job status <job>               # 失败 alloc 列表
nomad alloc status <alloc-id>        # 看 Recent Events:Setup Failure / Exit Code / OOM
nomad eval list -job=<job>           # 看是不是 placement 失败
```

**Recent Events 是判断分层的关键**:

- `Setup Failure` → Nomad 自己挂(磁盘 / 网络 / 驱动)
- `Started → Terminated Exit Code N` → 应用层挂,看 logs

### 阶段 2 — 应用层(任务起来后挂的)

```bash
nomad alloc logs <alloc-id>
nomad alloc logs -stderr <alloc-id>
```

找 FATAL / panic / connection refused / timeout 的目标地址。

### 阶段 3 — 验证目标依赖

```bash
# 直接探端口,不要猜
for ip in <list>; do
  timeout 2 bash -c "</dev/tcp/$ip/<port>" && echo OPEN || echo CLOSED
done
```

### 阶段 4 — 验证配置来源(关键!)

应用报错的地址 / 端口,**一定要回到 hcl 文件验证,并跨所有 server 对比**:

```bash
nomad job inspect <job> | grep <env-var>          # 部署里实际是什么
grep <env-var> /opt/app-deployment/jobs/*.hcl     # 文件里写的是什么
# 跨多台 server 比对,看是否漂移
for h in cntl1297 cntl1298 cntl1299; do
  ssh $h "grep <env-var> /opt/app-deployment/jobs/<job>.hcl"
done
```

## 六、长期预防建议

1. **把 `/opt/app-deployment/` 放进 git** — 这次的漂移持续了近 5 个月没人发现,git + CR 能直接消除这类问题
2. **统一部署入口** — 规定只在某一台(如 73)做 `nomad job run`,或写脚本强制从 git 拉取最新版
3. **74 上要么补齐目录、要么明确禁用部署**,避免下次手误又多一个漂移源
4. **加 healthcheck** — orchestrator 的 Nomad `check` 块如果配了 Redis 健康检查,这次 12:02 部署完会立刻被标 unhealthy,而不是反复 exit 1 直到被锁死
5. **定期 diff 跨节点的 hcl 文件**(可以写个 cron):

   ```bash
   for f in /opt/app-deployment/jobs/*.hcl; do
     md5sum $f
   done
   # 在 72/73/74 上跑,比对哈希是否一致
   ```
