# MinIO 对象存储部署 — AgentSphere 沙箱服务（私有云生产环境）

> **更新日期**: 2026-04-03
> **所属环境**: ISS 私有云生产集群
> **服务对象**: AgentSphere 沙箱平台（沙箱模板镜像存储、Firecracker rootfs/kernel 等）

## 基本信息

| 项目 | 值 |
|------|-----|
| 所属环境 | ISS 私有云生产集群 |
| 服务对象 | AgentSphere 沙箱平台（沙箱模板存储） |
| 部署节点 | server-54 (10.10.205.54) |
| API 端口 | 9000 |
| Console 端口 | 9001 |
| 容器名 | minio |
| 容器运行时 | Docker |
| 镜像版本 | `minio/minio:RELEASE.2023-03-20T20-16-18Z` |
| 数据目录 | `/orchestrator/minio/data`（独立 16TB 磁盘挂载 `/dev/loop20`） |
| Root 用户 | minioadmin |
| Root 密码 | minioadmin |

## 存储概览

| Bucket | 大小 | 用途 |
|--------|------|------|
| template | 9.1 TB | **主 Bucket**，AgentSphere 沙箱模板镜像（rootfs、kernel 等） |
| warp-benchmark-bucket | 空 | 性能测试用，可清理 |
| data | 空 | 未使用 |

| 磁盘 | 总容量 | 已用 | 可用 | 使用率 |
|------|--------|------|------|--------|
| `/dev/loop20` → `/orchestrator/minio/data` | 16 TB | 9.1 TB | 6.1 TB | 60% |

## 存储配置（16TB Loop 设备）

```bash
ssh root@10.10.205.54

# 创建目录
mkdir -p /orchestrator/minio

# 创建 16TB 稀疏文件（初始为 10TB，已扩容至 16TB）
truncate -s 16T /orchestrator/minio/minio-disk.img

# 格式化
mkfs.ext4 -F /orchestrator/minio/minio-disk.img

# 创建挂载点并挂载
mkdir -p /orchestrator/minio/data
mount -o loop /orchestrator/minio/minio-disk.img /orchestrator/minio/data

# 开机自动挂载
echo '/orchestrator/minio/minio-disk.img /orchestrator/minio/data ext4 loop,defaults 0 0' >> /etc/fstab

# 验证
df -h /orchestrator/minio/data
```

---

## 部署命令

```bash
ssh root@10.10.205.54

# 启动 MinIO
docker run -d \
  --name minio \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 9001:9001 \
  -v /orchestrator/minio/data:/data \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin \
  -e MINIO_REGION_NAME=us-west-1 \
  -e TZ=Asia/Shanghai \
  docker.1ms.run/minio/minio:RELEASE.2023-03-20T20-16-18Z \
  server /data --console-address ":9001"

# 验证
docker ps | grep minio
curl -f http://localhost:9000/minio/health/live
```

## 访问地址

- Web 控制台: http://10.10.205.54:9001
- API 端点: http://10.10.205.54:9000

## Bucket 创建

```bash
# 配置别名
mc alias set iss-private http://10.10.205.54:9000 minioadmin minioadmin
# 输出: Added `iss-private` successfully.

# 创建 Bucket
mc mb iss-private/template
# 输出: Bucket created successfully `iss-private/template`.

# 查看 Bucket 列表
mc ls iss-private
# 输出: [2026-01-28 17:25:59 CST]     0B template/

# 删除 Bucket（如需要）
# mc rb iss-private/template
```

## AKSK 创建

### 通过 mc 命令创建

```bash
# 配置别名
mc alias set iss-private http://10.10.205.54:9000 minioadmin minioadmin
# 输出: Added `iss-private` successfully.

# 创建 Service Account（自动生成 Access Key/Secret Key）
mc admin user svcacct add iss-private minioadmin
# 输出示例:
# Access Key: IEVNRUTRHVYUKT3MRWC1
# Secret Key: +JbO0gLw6dQlVFhUhasGhGWcgec5oiCefe7Qo2BE
# Expiration: no-expiry

# 列出所有 Service Accounts
mc admin user svcacct list iss-private minioadmin

# 删除 Service Account（如需要）
# mc admin user svcacct rm iss-private IEVNRUTRHVYUKT3MRWC1
```

### 通过 Web Console 创建

1. 访问 http://10.10.205.54:9001
2. 登录 minioadmin / minioadmin
3. 左侧菜单选择 "Access Keys"
4. 点击 "Create access key"
5. 保存生成的 Access Key 和 Secret Key

## mc 配置

```bash
mc alias set iss-private http://10.10.205.54:9000 minioadmin minioadmin
mc ls iss-private
```

## AWS CLI 配置

```bash
# 使用 Access Key/Secret Key
export AWS_ACCESS_KEY_ID=IEVNRUTRHVYUKT3MRWC1
export AWS_SECRET_ACCESS_KEY='+JbO0gLw6dQlVFhUhasGhGWcgec5oiCefe7Qo2BE'

aws s3 ls --endpoint-url http://10.10.205.54:9000

# 或使用 Root 凭证（不推荐）
# export AWS_ACCESS_KEY_ID=minioadmin
# export AWS_SECRET_ACCESS_KEY=minioadmin
```

## 常用运维命令

```bash
# 查看容器状态
docker ps | grep minio

# 健康检查
curl -sf http://10.10.205.54:9000/minio/health/live && echo "healthy" || echo "unhealthy"

# 查看磁盘使用
df -h /orchestrator/minio/data

# 查看各 Bucket 大小
du -sh /orchestrator/minio/data/template
du -sh /orchestrator/minio/data/*

# 查看容器日志
docker logs --tail 50 minio

# 重启服务
docker restart minio

# 检查 loop 设备
losetup -a | grep minio

# 检查端口占用
netstat -tlnp | grep -E '9000|9001'

# 进入容器
docker exec -it minio sh
```

---

## 存储扩容

```bash
# 1. 停止 MinIO
docker stop minio

# 2. 卸载当前存储
umount /orchestrator/minio/data

# 3. 扩容磁盘镜像（例如从当前 16TB 扩容到 20TB）
truncate -s 20T /orchestrator/minio/minio-disk.img

# 4. 调整文件系统大小
e2fsck -f /orchestrator/minio/minio-disk.img
resize2fs /orchestrator/minio/minio-disk.img

# 5. 重新挂载
mount -o loop /orchestrator/minio/minio-disk.img /orchestrator/minio/data

# 6. 启动 MinIO
docker start minio

# 7. 验证
df -h /orchestrator/minio/data
```

---

## 备份

```bash
# 备份 MinIO 全部数据
rsync -avz --progress \
  /orchestrator/minio/data/ \
  /backup/minio-$(date +%Y%m%d)/

# 仅备份 template bucket
rsync -avz --progress \
  /orchestrator/minio/data/template/ \
  /backup/minio-template-$(date +%Y%m%d)/
```

---

## 故障排查

```bash
# 检查容器日志
docker logs --tail 100 minio

# 检查存储挂载
df -h /orchestrator/minio/data
mount | grep minio

# 检查端口占用
netstat -tlnp | grep -E '9000|9001'

# 检查 loop 设备
losetup -a | grep minio

# 进入容器排查
docker exec -it minio sh
```

---

## 注意事项

1. **版本较旧**：当前运行 2023-03-20 版本，建议评估升级
2. **默认密码**：使用默认凭证 minioadmin/minioadmin，生产环境建议修改
3. **无冗余**：单节点部署，标准奇偶校验为 0，建议配置备份策略
4. **存储监控**：当前使用率 60%（9.1TB/16TB），定期检查，及时扩容或清理
5. **自动挂载**：Loop 设备已配置 `/etc/fstab` 开机自动挂载

---

**维护**: AgentSphere Team
**最后更新**: 2026-04-03
