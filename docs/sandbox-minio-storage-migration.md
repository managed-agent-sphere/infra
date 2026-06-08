# MinIO 沙箱存储清理纪要

时间：2026-05-18
环境：agentsphere 生产集群（10.10.205.0/24 网段）

## 一、起因

54 机器上 MinIO 的数据盘到 90%：

```
$ df -h /orchestrator/minio/data
/dev/loop20      16T   14T  1.7T   90% /orchestrator/minio/data
```

再涨下去新沙箱的快照上传会失败，pause 操作也可能受影响，需要立刻处理。

### 先说结论

整理分析观察：

  - MinIO 14T 里 12.5T 都是 memfile。要省空间必须动 memfile 。

  - snapshot-cleaner 的 truncate memfile 策略已经把大部分能省的都省了。当前剩下的 12 TB memfile (7600 多个引用)基本都是真用户的 paused 状态数据，没有"白占"的余地。

  - 这套 layered 设计是 e2b 上游一直没解决的问题，他们自己的代码里就挂着 ENG-3477 这个工单。我们不要自己造轮子。

  - ext4 4K block 的 16T 单文件上限是个硬约束，直接扩 image 这条路堵死了。

  - 真正持续控制存储增长的，最后还是要业务侧定一个暂停沙箱的保留期。

## 二、系统结构

业务层有两套系统：codesphere 是面向用户的业务层，agentsphere 是底层基础设施。两边各有一个 PostgreSQL：

- codesphere 库在 10.10.206.206，里面有 sandboxes / projects / users 这些业务表
- agentsphere 库在 10.10.205.49，里面有 instances / snapshots / envs / env_builds 等

用户在 codesphere 创建沙箱，codesphere 调用 agentsphere 的 API 真正起一个 VM，最终的内存快照、磁盘镜像存到 54 机器上的 MinIO 里。两边通过 sandbox 的 id 关联。

54 上 MinIO 的存储栈是这样的：

```
/orchestrator/minio/data            （MinIO 服务读写的目录）
        ↑ mount
/dev/loop20                          （loop 块设备，16T）
        ↑ backing
/orchestrator/minio/minio-disk.img   （一个 16T 的磁盘镜像）
        ↑ 物理写入
/dev/sda                             （21T 物理盘，ext4）
```

几个有关清理的细节：
- MinIO 是单实例单盘模式
- 没开 versioning，也没有 lifecycle policy
- backing image 是 sparse 文件，外面看 16T，实际只占 14T

## 三、当前数据规模

简单统计了一下两边表：

- envs：3411 行。这个数字大部分不是"用户主动创建的模板"。每个暂停沙箱在第一次 pause 时会生成一个 env 作为这个沙箱专属的"快照容器"，之后反复 pause/resume 都共用这同一个 env。这 3411 行拆开看：约 3192 行对应下面 3192 个暂停沙箱的快照容器，25 个左右是真正的 base 模板（被 snapshots.base_env_id 引用的），245 个是没人引用的残留
- env_builds 中已经上传到 MinIO 的：11536 个。**每次 pause 都会在 env_builds 表新增一行**（同一沙箱反复 pause 也会一直累积），所以这个数字远大于沙箱数。11536 / 3192 ≈ 3.6，意思是平均每个暂停沙箱挂着 3.6 个历史 build——这就是后面讲的"累积链"问题
- snapshots（暂停状态的沙箱）：3192 个。**一个暂停沙箱一行**，反复 pause/resume 不会新增 snapshots 表的行（只会更新原行）
- codesphere.sandboxes（用户视角的沙箱）：6285 个

MinIO 里 13.57 TiB 数据按类型拆开：

- memfile：12.5 TiB，占 92%
- rootfs.ext4：1.04 TiB，占 8%
- 其它（header、snapfile、metadata）：合起来不到 3 GiB

memfile 占了绝大部分，这是后面分析的重点。

## 四、问题分析

### 4.1 为什么会有这么多数据

沙箱有一个 pause / resume 的设计：用户不活动一段时间会自动 pause，把 VM 状态打成快照存到 MinIO；下次访问 resume 几秒钟内就能恢复。这个设计是为了控制冷启动延迟，代价就是吃存储。

每次 pause 平均产生一个 1 到 2G 的新 build：memfile 通常 1G 多，rootfs 增量几 MB 到几十 MB。同一个沙箱反复 pause / resume，build 就一直累积。

更麻烦的是，新生成的 build 不是独立的快照，而是增量 —— 它的 header 文件里记着一张块映射表，把"哪段数据在哪个 build 的存储里"标得清清楚楚。所以新 build 的某些块实际数据是引用旧 build 的，少了任意一个被引用的祖先 build，新 build 都没法 resume。

实测一个反复 pause 三次的沙箱：

```
第一次 pause（gen 3）   1.7 GiB （自己的数据）
第二次 pause（gen 4）   1.4 GiB （+ 引用 gen 3 的 358 MiB memfile）
第三次 pause（gen 5）   1.3 GiB （+ 引用 gen 3、4）
                       --------
                       共 4.4 GiB
```

链是真累积式的，每次 pause 上一代的依赖不会被合并进新 build。

### 4.2 孤儿沙箱从哪来的

最早 `agentsphere sbx list | wc -l` 和 `agentsphere sbx list -s paused | wc -l` 看下来一共 6000 多沙箱，但相当一部分跟用户对不上：

- 用户在 codesphere 删了自己或者删了项目，sandboxes 行通过外键级联也删了，但 codesphere 这边没去调 agentsphere 的 kill 接口，结果 agentsphere 那边的 snapshots、envs、env_builds 还有 MinIO 上的文件都留下来了
- 有些沙箱的 metadata 里压根没 user_id，是早期或者非业务路径创建的
- warm pool 里预热但没被分配出去的

这几种加起来大概一千多个，是典型的孤儿。

### 4.3 instances 表残留

`agentsphere sbx kill` 在清暂停沙箱时，cascade 删了 envs/env_builds/snapshots，也走 template-manager 删了 MinIO 文件，但是 instances 表里那一行没动。所以即便沙箱实际清干净了，instances 表里还留着记录。

## 五、清理过程

清理脚本主要有四个，分别处理不同类型的孤儿：

- `cleanup-no-user.sh`：清理 metadata 里没有 user_id 的沙箱
- `cleanup-warm.sh`：清理 warm pool 里的预热沙箱
- `cleanup-db-orphans.sh`：清理 agentsphere 那边还有但 codesphere 已经不认的暂停沙箱，并且补一刀直接 SQL 把 instances 残留也清掉
- `snapshot-cleaner`：处理累积的旧 build，下一节单独说

前三个加上 snapshot-cleaner 一起跑下来，删了 3000 多个沙箱，腾出大约 1 TiB 空间。考虑到很多沙箱其实只在 agentsphere 数据库里有记录、MinIO 上的文件早就被前一轮清理跑掉了，最终落到盘上的释放量在 1T 这个量级是合理的。

## 六、snapshot-cleaner 的清理逻辑

这是 agentsphere 自带的一个工具，处理的是"累积链 + 失败 build"。它的逻辑值得展开说一下，因为是后续运维的核心。

工具分三层保护规则，然后给每个 build 决定要怎么处理：

保护规则部分：

1. 公开模板（env_aliases 表里有别名的）的最新 build 必须留，并且把它 header 里引用到的祖先 build 也加进保护集
2. 每个暂停沙箱保留最新 1 个 build（生产环境改过的，默认是 2 个），把这个 build 的 memfile.header 和 rootfs.ext4.header 分别解析，把引用到的 build 分别加进 memfileDeps 和 rootfsDeps 集合
3. 如果 header 读不到（文件损坏、网络问题等），保险起见把这个沙箱的所有 build 全保护起来

处理动作部分，对 MinIO 上的每一个 build：

- 在 fullyProtected 集合里：完整保留
- 在 memfileDeps 集合里：memfile 还被引用，整个 build 保留
- 只在 rootfsDeps 集合里（memfile 没被引用了）：把 memfile 截成 0 字节，rootfs 留着
- 都不在：整个 build 目录删掉

最聪明的是 truncate memfile 这条。memfile 通常 1.5 GiB 上下，rootfs 只有几兆到几十兆。一个旧 build 如果 rootfs 还被新 build 用着，memfile 却被新 pause 完全覆盖过了，那就只清 memfile，rootfs 留下来给新 build 接着引用。

这套机制实测已经在跑了，当前 14T 数据是已经是处理后的数据。

## 七、扩容路径上的问题

最初想的方案是把 MinIO 的 image 文件从 16T 扩到 18 或 19T，sda 还有 6.2T 物理空间。结果一执行就撞墙：

```
truncate: 在 19791209299968 字节处截断失败: 文件过大
```

原因是 sda 用的是 ext4，4K block，单文件理论上限 (2^32 - 1) × 4096 = 15.99 TiB。当前 image 已经 15.999 TiB，几乎贴到上限了，不能再用 truncate 扩。

绕过 16T 限制的几种路径：

- 加新物理盘，单独挂载，单独建 fs
- 把 sda 重新格式化成 xfs（单文件上限 8 EiB），代价是 14T 数据要先迁出再迁回，sda 自己没那么多空闲空间能放
- 用 device-mapper 把多个 image 拼成一个大块设备（运行时一半数据在网络上，故障域大，不推荐）
- 整体迁移到 56 机器（56 上 sda1 是 38T，只用了 1%）

迁移到 56 这条最干净，但是 54 和 56 之间是千兆网络，14T 全量传过去要 37 小时左右。如果要快，得加 10G 网卡或者直连 DAC。

## 八、长期建议

按优先级和投入产出来排：

**业务层 TTL 政策**：这是不动代码就能释放 TB 级空间的唯一路径。按当前快照年龄分布，超过 30 天没被 resume 的暂停沙箱有 1213 个，对应大约 1.8 TiB；超过 90 天的 150 个，大约 225 GiB。codesphere 产品决定一个保留期，到期前通知用户，到期自动调 `agentsphere sbx kill` 就能回收。这是产品决策，技术上几十行脚本就能做。

**memfile 压缩**：pause 时 zstd 或者 lz4 压缩，uffd handler 解压。VM 内存里大量是 zero page 和重复模式，估算 3 到 5 倍压缩率，能省 8 到 10 TiB。工程量中等，但 ROI 最高。

**加新物理盘**：硬件流程慢，但走通之后是最干净的方案，新盘上可以直接用 xfs 不用受 16T 限制。

**迁移到 56 + xfs**：长期最理想，但要先把 54-56 升到 10G 网络，不然光迁移就要一两天。


