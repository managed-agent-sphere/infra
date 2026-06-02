# 修改 SANDBOX_ALLOWED_IPS 操作手册

## 作用

`SANDBOX_ALLOWED_IPS` 是一个 IP/CIDR 白名单，决定 sandbox 内部能访问哪些外部 IP。

> ⚠️ 只对**修改之后新建的 sandbox** 生效，老 sandbox 不变。

## 位置

- 机器：`10.143.181.73`
- 文件：`/opt/app-deployment/jobs/orchestrator.hcl`
- 变量：`SANDBOX_ALLOWED_IPS`（约第 82 行）

格式：英文逗号分隔，**不加空格**

```hcl
SANDBOX_ALLOWED_IPS = "10.x.x.x,10.10.205.50,192.168.1.0/24"
```

## 修改步骤

全部在 **10.143.181.73** 上执行。

### 1. 备份

```bash
cd /opt/app-deployment/jobs
cp -a orchestrator.hcl orchestrator.hcl.bak.$(date +%Y%m%d-%H%M%S)
```

### 2. 编辑

```bash
vim /opt/app-deployment/jobs/orchestrator.hcl
```

找到 `SANDBOX_ALLOWED_IPS = "..."`，改成你要的值。

### 3. 预演（确认改动正确）

```bash
nomad job plan /opt/app-deployment/jobs/orchestrator.hcl
```

应该看到：

```
+/- Env[SANDBOX_ALLOWED_IPS]: "<旧值>" => "<新值>"
...
Job Modify Index: 12345
```

确认 diff **只有 SANDBOX_ALLOWED_IPS 这一行**。记下 `Job Modify Index` 数字。

### 4. 执行

```bash
nomad job run -check-index <上一步的数字> /opt/app-deployment/jobs/orchestrator.hcl
```

### 5. 等滚动完成（2-3 分钟）

```bash
watch -n 2 'nomad job status orchestrator | tail -15'
```

直到所有 alloc 都是 `running` 且版本号统一就好了。

### 6. UI 验证

打开浏览器 http://10.143.181.73:4646/ui/jobs/orchestrator/definition?view=job-spec
查看：SANDBOX_ALLOWED_IPS 是否是期望的

## 回滚

如果出问题：

```bash
cd /opt/app-deployment/jobs
# 用刚才的备份还原
cp -a orchestrator.hcl.bak.<时间戳> orchestrator.hcl
# 重新部署
nomad job plan /opt/app-deployment/jobs/orchestrator.hcl
nomad job run -check-index <新的index> /opt/app-deployment/jobs/orchestrator.hcl
```

## 常见问题

**老 sandbox 没生效？** 新的修改理论上作用的新建的沙箱。

**白名单加了 IP 还是连不上？**
1. 确认是修改后**新建**的 sandbox（不是旧的）
2. 目标 IP 那台机器自己的防火墙也要允许 sandbox 来访
