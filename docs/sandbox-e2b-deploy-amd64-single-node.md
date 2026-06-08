# E2B 单机部署端口说明

本文记录当前 managed-agent-sphere / installer 单机部署里的端口分配，以及它和 e2b-dev/infra 上游 IaC 的差异。

## 当前单机部署端口

当前不是多个 API / orchestrator / template-manager 进程，而是每个服务进程会监听多个端口。

| 端口 | 进程 | 作用 | 说明 |
| ---: | --- | --- | --- |
| `50001` | `api` | HTTP REST API / health | installer 自定义 `API_PORT`。|
| `5009` | `api` | API internal gRPC | 保持上游原生 `api_internal_grpc_port=5009`。|
| `5109` | `api` | API edge gRPC | 源码默认 `API_EDGE_GRPC_PORT=5109`；上游 Nomad job 通常用动态端口。|
| `5008` | `orchestrator` | orchestrator 主服务端口 | gRPC 和 HTTP health 通过 cmux 共用一个端口；保持上游原生 `orchestrator_port=5008`。|
| `5007` | `orchestrator` | sandbox proxy | 上游原生 `orchestrator_proxy_port=5007`。|
| `5010` | `orchestrator` | sandbox hyperloop/internal proxy | orchestrator 源码默认 `SANDBOX_HYPERLOOP_PROXY_PORT=5010`。|
| `5012` | `template-manager` | template-manager 主服务端口 | gRPC 和 HTTP health 通过 cmux 共用；单机部署避让端口。|
| `15007` | `template-manager` | build sandbox proxy | installer 配置 `PROXY_PORT=15007`。|
| `5011` | `template-manager` | build sandbox hyperloop/internal proxy | installer 配置 `SANDBOX_HYPERLOOP_PROXY_PORT=5011`。|
| `3002` | `client-proxy` | sandbox traffic HTTP proxy | 新版 client-proxy 读取 `PROXY_PORT`，当前为 `3002`。|
| `3003` | `client-proxy` | health check | 新版 client-proxy 读取 `HEALTH_PORT`，当前为 `3003`。|

当前远端验证时看到的监听关系如下：

```text
api pid=89807:
  :50001  HTTP REST API / health
  :5009   API internal gRPC
  :5109   API edge gRPC

orchestrator pid=22713:
  :5008   orchestrator gRPC + HTTP health
  :5007   sandbox proxy
  :5010   sandbox hyperloop/internal proxy

template-manager pid=89221:
  :5012   template-manager gRPC + HTTP health
  :15007  build sandbox proxy
  :5011   build sandbox hyperloop/internal proxy

client-proxy:
  :3002   sandbox traffic HTTP proxy
  :3003   health check
```

## 上游 IaC 默认端口

从本地上游源码 `/Users/axp/stone/e2b-dev/infra` 看，关键默认值如下：

| 变量 | 默认端口 | 文件 |
| --- | ---: | --- |
| `api_internal_grpc_port` | `5009` | `iac/provider-aws/nomad/variables.tf` |
| `orchestrator_port` | `5008` | `iac/provider-aws/nomad/variables.tf` |
| `orchestrator_proxy_port` | `5007` | `iac/provider-aws/nomad/variables.tf` |
| `template_manager_port` | `5008` | `iac/provider-aws/nomad/variables.tf` |

注意：上游 IaC 里 `orchestrator_port` 和 `template_manager_port` 都默认是 `5008`。这不是错误，因为上游按不同 node pool 部署：

```text
api              -> api_node_pool
orchestrator     -> orchestrator_node_pool
template-manager -> build_node_pool
```

因此上游允许不同服务在不同机器上复用相同 host port。

## 单机部署为什么需要避让

当前 installer 是单机 Nomad，所有 job 都跑在同一个 `default` node pool / 同一台 host 上。单机 host port 不能重复，因此不能完全照搬上游端口布局。

之前 installer 曾把：

```text
ORCHESTRATOR_PORT=5008
TEMPLATE_MANAGER_PORT=5009
```

当 API 补上上游原生的：

```text
API_INTERNAL_GRPC_PORT=5009
```

就会出现 `api` 和 `template-manager` 抢 `5009` 的冲突。

当前修正策略：

```text
API internal gRPC 保持上游原生 5009
orchestrator 主端口保持上游原生 5008
template-manager 主端口在单机部署中避让到 5012
```

这样保留了 API / orchestrator 的关键上游端口，同时让 template-manager 通过 Consul 地址暴露给 API：

```text
TEMPLATE_MANAGER_HOST=template-manager.service.consul:5012
```

## client-proxy 新旧配置差异

当前使用的新镜像 `fork-20260526_104100` 中，client-proxy 不再读取旧配置里的 `EDGE_PORT` 作为健康端口。源码配置为：

```text
HEALTH_PORT 默认 3003
PROXY_PORT  默认 3002
```

因此 Nomad job 的 health check 应该打 `3003`，sandbox traffic proxy 应该走 `3002`。旧服务器 48 上的 `client-proxy:20260119_104511` 仍使用旧式 `EDGE_PORT=3001` 配置，不能直接套到当前新镜像。

## template-manager 调用频率

`template-manager` 不是普通 sandbox 启动链路里的每次高频调用对象。它主要处理 template / environment build 相关工作：

- 创建 template / environment build。
- 查询和同步 build 状态。
- 上传或处理 template layer 文件。
- 写入构建产物、缓存和模板存储。

因此：

- 大量运行已有 template 的 sandbox 时，主要压力在 `api`、`orchestrator`、sandbox proxy、存储/cache。
- 大量并发构建 template 时，`template-manager` 会成为重负载服务，可能消耗 CPU、IO、网络和存储带宽。
- 上游把 template-manager 放在 `build_node_pool`，并配有 autoscaler，正是为了把 build 类负载和 API / orchestrator 分开扩缩容。

## 相关源码位置

API 监听 HTTP、internal gRPC、edge gRPC：

```text
/Users/axp/stone/e2b-dev/infra/packages/api/main.go
/Users/axp/stone/e2b-dev/infra/packages/api/internal/cfg/model.go
```

上游 API Nomad job：

```text
/Users/axp/stone/e2b-dev/infra/iac/modules/job-api/jobs/api.hcl
/Users/axp/stone/e2b-dev/infra/iac/modules/job-api/variables.tf
```

上游 orchestrator Nomad job：

```text
/Users/axp/stone/e2b-dev/infra/iac/modules/job-orchestrator/jobs/orchestrator.hcl
```

上游 template-manager Nomad job：

```text
/Users/axp/stone/e2b-dev/infra/iac/modules/job-template-manager/jobs/template-manager.hcl
```

上游 provider 里 node pool 和默认端口：

```text
/Users/axp/stone/e2b-dev/infra/iac/provider-aws/nomad/main.tf
/Users/axp/stone/e2b-dev/infra/iac/provider-aws/nomad/variables.tf
```
