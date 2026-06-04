# 集群故障排查报告 2026-06-03

## 一、沙箱全量落到 sbx-243，sbx-14 无法调度

### 根因

1. **sbx-14 心跳丢失**：2026-06-02 17:34 Nomad 记录 heartbeat missed，4 秒后重连。
2. **API 标记节点失败**：约 18:06 orchestration-api 把 sbx-14 写入 Redis `node:failed:87da636d-...`，TTL 24h。
3. **调度器跳过失败节点**：BestOfK 算法检查 `node:failed:*` 并跳过，后续所有沙箱全部落到 sbx-243。
4. **鸡生蛋问题**：`node:registry:87da636d-...` 键不存在，API 健康检查只读不写新节点，sbx-14 永远无法自动补录。

### 修复方法

```bash
# 1. drain 节点（节点已空，瞬间完成）
nomad node status self
nomad node drain -enable -deadline 5m -yes 87da636d

# 2. 重启宿主机（清 706 个孤儿 netns 和 stale 状态）
ssh 10.142.70.14 reboot

# 3. 重启后发现 systemd-resolved 被 disable，DNS 解析失败
ssh 10.142.70.14 systemctl enable --now systemd-resolved

# 4. 恢复调度资格
nomad node eligibility -enable 87da636d

# 5. 重调度 system job
nomad job eval -force-reschedule orchestrator
nomad job eval -force-reschedule template-manager

# 6. 清除 Redis 失败标记
redis-cli DEL node:failed:87da636d-f617-ea7b-8cd9-c79db45e9080

# 7. 手动写入节点注册（鸡生蛋问题）
redis-cli SET node:registry:87da636d-f617-ea7b-8cd9-c79db45e9080 10.142.70.14
```

---

## 二、sbx-14 重启后 orchestrator 无法启动（DNS 解析失败）

### 根因

- `/etc/systemd/resolved.conf.d/custom-dns.conf` 的 `Domains=~consul` 只把 `.consul` 域路由到内网 DNS。
- `*.iss-ruidong.com` 走链路默认上游（阿里云公网 DNS），解析到错误的公网 IP `8.141.111.57`（SLB，HTTP 访问返回 302 重定向）。
- orchestrator 启动时连接 Redis（域名 `r-2zekupw72tl9ln4qqc.redis.rds.aliyuncs.com`）失败，进程 FATAL 退出。
- `10.142.10.93` 上运行着 dnsmasq，内置了 `*.iss-ruidong.com → 10.142.40.246` 的映射，需要路由过去。

### 修复方法

在所有节点 `/etc/systemd/resolved.conf.d/custom-dns.conf` 添加 `Domains=~iss-ruidong.com`：

```ini
[Resolve]
DNS=10.142.10.93
Domains=~iss-ruidong.com
FallbackDNS=8.8.8.8
```

```bash
systemctl restart systemd-resolved
# 验证：dig +short api.iss-ruidong.com @127.0.0.53 → 应返回 10.142.40.246
```

**已修复节点：** 10.142.70.15  
**待修复节点：** 10.142.70.14、10.142.40.242、10.142.40.243（同样需要追加该配置）

---

## 三、302 错误："获取沙箱状态失败: 302: b''"

### 根因

codesphere → AgentSphere SDK → `api.iss-ruidong.com` 解析到公网 IP（SLB），HTTP 访问被强制重定向到 HTTPS，返回 302 + 空 body。根本原因同上：节点 DNS 配置缺少 `Domains=~iss-ruidong.com` 路由规则。

---

## 四、沙箱 isbhcntx988g72yymnw6u 无法 resume（envd 不响应）

### 根因

1. 该沙箱在某次 resume 过程中（高负载期间），pause 拍 snapshot 时 envd 恰好处于网络 I/O 等待中。
2. resume 后 envd 继续等待已失效的 TCP 连接，其 event loop 被阻塞。
3. orchestrator 每 50ms 向 `http://VM_IP:49983/init` POST 请求，envd 不响应，60s 超时后沙箱被 kill。

### 关键代码路径

```
sandbox.go: WaitForEnvd(ctx, 60s)
  └─ envd.go: doRequestWithInfiniteRetries → POST /init 每 5ms 重试，每次 50ms 超时
       └─ 60s 后 ctx.Done() → 沙箱被 cleanup
```

**没有 envd 重启 fallback**，orchestrator 只能放弃。

### 修复过程

#### 方案 Y（成功）：回退到上一个健康的 env_build

```sql
-- 将损坏/可疑的 build 标记失败
UPDATE env_builds SET status = 'failed'
WHERE id IN (
  'db0a6b25-4b26-4010-9932-a92ef61e613a',  -- envd 卡死
  '2b0f0ab6-f6fe-4207-b312-ec8293ce6065',  -- Z 方案假 build
  '22e9327b-a2c7-4956-b341-129a211cf85f',  -- Z 污染
  '0224879a-71f0-4cf0-b6ee-13c903a55809'   -- Z 污染
);
-- resume 的 SQL: ORDER BY finished_at DESC，自动选到 c713a01c（2026-06-02 11:29）
```

#### 方案 Z（失败，记录教训）：替换 memfile 为 base 模板的 fresh-boot 状态

- **预期**：用 base 的 snapfile+memfile（fresh envd）+ user 的 rootfs（文件保留）= 虚拟重启
- **实际**：base 的 memfile 包含 base 内核的 VFS/inode 缓存，与 user rootfs 的 inode 表不一致，导致 EXT4 报错、文件丢失

```
dmesg: EXT4-fs error: deleted inode referenced
```

**结论：snapfile + memfile 必须是同一时刻共同生成的，不能分开替换。**

---

## 五、"重启沙箱保留文件"的正确实现方式（未来改进）

### 现有问题

当前 orchestrator 没有"用用户 rootfs 冷启动并重拍 snapshot"的路径。只有：
1. 从 base 模板冷启动（丢用户文件）
2. 从 paused snapshot resume（需要 envd 健康）

### 推荐新增操作

```
ForceReboot(sandboxID):
  1. 获取用户最新 env_build 的 rootfs.ext4 + header
  2. 用 base kernel 冷启动 VM，挂载用户 rootfs 作为 block device
  3. 等待 envd healthy
  4. 拍 snapshot → 新 env_build（snapfile+memfile 与 user rootfs 匹配）
  5. 存 OSS，写 DB，旧 build 标为 superseded
```

这样可以在 envd 卡死时保留用户文件并恢复可用性，代价仅是 RAM 状态丢失（进程重启）。

---

## 六、快速排查备忘

### 确认沙箱节点分布

```bash
# Redis 查所有活跃沙箱及其节点
redis-cli KEYS 'orchestrator:sandbox:*' | while read k; do
  redis-cli GET $k | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['sandbox_id'], d['node_id'])"
done
```

### 确认节点是否被标记失败

```bash
redis-cli KEYS 'node:failed:*'
redis-cli TTL 'node:failed:<nodeUUID>'
```

### 强制注册节点

```bash
redis-cli SET node:registry:<nodeUUID> <nodeIP>
```

### 查 env_build 历史（按时间倒序）

```sql
SELECT id, status, finished_at FROM env_builds
WHERE env_id = '<envID>'
ORDER BY finished_at DESC LIMIT 10;
```

### 修改 resume 目标节点

```sql
UPDATE snapshots SET origin_node_id = '<targetNodeUUID>'
WHERE sandbox_id = '<sandboxID>';
```

---

## 七、涉及集群节点

| 节点 IP | 角色 | 说明 |
|---|---|---|
| 10.142.10.93 | server-93 / jit | Nomad server + client，dnsmasq（`:53`），Consul server |
| 10.142.70.15 | server-15 | Nomad server，api / client-proxy alloc |
| 10.142.40.242 | server-242 | Nomad server |
| 10.142.70.14 | sandbox-14 (sbx-14) | Nomad client，sandbox node |
| 10.142.40.243 | sandbox-243 (sbx-243) | Nomad client，sandbox node（主力节点） |
| 10.142.40.246 | — | api.iss-ruidong.com 指向 IP（内网 API 入口）|
