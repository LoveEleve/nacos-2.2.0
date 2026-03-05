# 第13章：高频面试题汇总

> **源码版本**：Nacos 2.x  
> **覆盖章节**：第1章～第12章全部核心知识点  
> **难度标注**：⭐ 基础 | ⭐⭐ 中等 | ⭐⭐⭐ 高频核心

---

## 使用说明

本章将前12章的核心知识点提炼为面试题形式，每题包含：
- **考察点**：面试官真正想考什么
- **标准答案**：源码级别的精准回答
- **追问方向**：面试官可能的追问

---

## 第一部分：配置中心（Config）

### Q1：Nacos 配置中心如何实现配置的实时推送？⭐⭐⭐

**考察点**：长轮询原理、1.x vs 2.x 的推送机制差异

**标准答案**：

**1.x 长轮询机制**（`LongPollingService.java`）：

```
① 客户端发起 HTTP 请求，携带本地所有配置的 MD5 列表
   Header: Long-Pulling-Timeout: 30000（ms）

② 服务端比对 MD5，若有变更立即返回变更的 dataId 列表

③ 若无变更，服务端挂起请求（timeout = 30000 - 500 = 29500ms）
   加入 allSubs 队列，等待配置变更事件

④ 配置变更时，触发 LocalDataChangeEvent
   DataChangeTask 遍历 allSubs，找到订阅该配置的客户端，立即响应

⑤ 超时（29.5秒）后返回空响应，客户端重新发起长轮询
```

**关键源码**（`LongPollingService.addLongPollingClient()`）：
```java
int delayTime = SwitchService.getSwitchInteger(SwitchService.FIXED_DELAY_TIME, 500);
// timeout = 客户端传入的30000 - 500 = 29500ms
long timeout = Math.max(10000, Long.parseLong(str) - delayTime);
```

**2.x gRPC 主动推送**（`RpcConfigChangeNotifier.java`）：
```
① 客户端通过 gRPC 长连接订阅配置
② 配置变更后，服务端通过 gRPC 双向流主动推送 ConfigChangeNotifyRequest
③ 客户端收到推送后，主动发起 ConfigQueryRequest 拉取最新配置内容
```

**对比**：

| 维度 | 1.x 长轮询 | 2.x gRPC 推送 |
|------|-----------|--------------|
| 推送延迟 | 最大 500ms（delayTime） | 毫秒级 |
| 连接数 | 每次轮询新建 HTTP 连接 | 复用 gRPC 长连接 |
| 服务端资源 | 线程挂起，占用 Servlet 线程 | 异步推送，无线程阻塞 |

**追问**：为什么是 29.5 秒而不是 30 秒？
> 提前 500ms 响应，避免客户端 HTTP 超时（客户端超时设置为 30 秒，服务端提前 500ms 返回，留出网络传输时间）。

---

### Q2：Nacos 配置 Dump 机制是什么？为什么需要三层存储？⭐⭐⭐

**考察点**：配置存储架构、三层存储的作用

**标准答案**：

**三层存储架构**：

| 层次 | 存储内容 | 作用 |
|------|---------|------|
| **DB**（Derby/MySQL） | 配置完整内容 | 持久化，数据不丢失 |
| **磁盘文件** | 配置完整内容 | 集群模式下供 HTTP 长轮询读取，DB 故障时降级 |
| **内存 CacheItem** | 仅 MD5 + 元数据 | O(1) 判断配置是否变更，避免每次查 DB |

**Dump 触发时机**（`DumpService.java`）：
- 启动时**全量 dump**（`DumpAllTask`）：将 DB 中所有配置同步到内存和磁盘
- 配置变更时**增量 dump**（`DumpTask`）：单条配置变更后立即同步
- 定时**全量 dump**（每 6 小时）：兜底保证一致性

**⭐ 重要细节**：standalone + Derby 模式**不写磁盘**（`PropertyUtil.isDirectRead()` 为 true），因为 Derby 本身就是持久化的，磁盘文件是为集群模式的 HTTP 长轮询服务的。

**追问**：DB 故障时 Nacos 还能提供配置服务吗？
> 能。服务端从磁盘文件读取配置（降级），客户端也有本地快照缓存（`~/.nacos/config/`），双重保障。

---

### Q3：Nacos 集群中配置如何同步？⭐⭐

**考察点**：集群配置同步机制、AsyncNotifyService

**标准答案**：

```
① Leader/任意节点收到配置变更请求
   ConfigOperationService.publishConfig() 写入 DB

② 发布 ConfigDataChangeEvent 事件

③ AsyncNotifyService 监听事件，异步通知集群所有节点
   - HTTP 方式：POST /v1/cs/communication/dataChange
   - gRPC 方式：ConfigChangeClusterSyncRequest

④ 各节点收到通知后，执行 DumpTask
   从 DB 读取最新配置，更新本地磁盘和内存缓存

⑤ 各节点内存 CacheItem 更新后，通知本节点的订阅客户端
```

**注意**：配置数据通过 **JRaft（CP 协议）** 保证强一致性，不是 Distro。

**追问**：如果某个节点 dump 失败怎么办？
> `AsyncNotifyService` 有重试机制，失败后会重新加入通知队列，最终保证一致性。

---

### Q4：Nacos 灰度发布（Beta 配置）是如何实现的？⭐

**考察点**：灰度发布原理

**标准答案**：

- 灰度配置存储在 `config_info_beta` 表，包含 `beta_ips` 字段（灰度 IP 列表）
- 客户端请求时携带自身 IP
- 服务端判断：若客户端 IP 在 `beta_ips` 中，返回灰度配置；否则返回正式配置
- 灰度配置发布后，可通过控制台"停止灰度"或"全量发布"操作

---

### Q5：Nacos 配置中心的 GroupKey 是什么？⭐

**考察点**：配置数据模型

**标准答案**：

```
GroupKey = dataId + "+" + group + "+" + tenant（namespace）
```

- `dataId`：配置 ID，通常是应用名 + 配置文件名（如 `order-service.yaml`）
- `group`：分组，默认 `DEFAULT_GROUP`，可用于区分环境或业务线
- `tenant`：命名空间 ID，用于环境隔离（dev/test/prod）

GroupKey 是配置的唯一标识，内存 CacheItem 以 GroupKey 为 key 存储 MD5。

---

### Q6：Nacos 客户端配置拉取失败如何降级？⭐⭐

**考察点**：客户端容灾机制

**标准答案**：

**两级降级**：

1. **本地快照（Snapshot）**：
   - 路径：`${user.home}/nacos/config/{namespace}/{group}/{dataId}`
   - 每次成功拉取配置后，客户端自动写入本地快照
   - 服务端不可用时，从快照加载（`LocalConfigInfoProcessor`）
   - 可通过 `SnapShotSwitch.setIsSnapShot(false)` 关闭快照功能

2. **本地覆盖（Override）**：
   - 路径：`${user.home}/nacos/config/{namespace}/config-data-tenant/{group}/{dataId}`
   - 优先级最高，可用于紧急覆盖线上配置

**优先级**：本地覆盖 > 服务端配置 > 本地快照

---

### Q7：Nacos 配置变更后，客户端多久能感知到？⭐⭐

**考察点**：配置推送延迟分析

**标准答案**：

**1.x 长轮询**：最大延迟 = `delayTime`（500ms）
- 配置变更触发 `DataChangeTask`，立即响应挂起的长轮询请求
- 客户端收到响应后，立即发起配置拉取请求
- 总延迟：通常 < 1 秒

**2.x gRPC 推送**：毫秒级
- 配置变更 → `RpcConfigChangeNotifier` 主动推送 → 客户端收到通知 → 拉取最新配置
- 总延迟：通常 < 100ms

**最坏情况**（服务端宕机重启）：
- 客户端重连后，重新订阅，服务端推送最新配置
- 重连期间使用本地快照

---

### Q8：Nacos 配置中心和 Spring Cloud Config 的区别？⭐

**考察点**：技术选型对比

**标准答案**：

| 对比项 | Nacos Config | Spring Cloud Config |
|--------|-------------|---------------------|
| 推送机制 | 长轮询/gRPC 主动推送 | 需要 Spring Cloud Bus + MQ |
| 实时性 | 高（< 1s） | 依赖 MQ，延迟较高 |
| 存储 | 内置 Derby/MySQL | Git 仓库 |
| 版本管理 | 内置历史记录 | Git 天然版本管理 |
| 部署复杂度 | 低（单独部署） | 高（需要 MQ） |
| 功能集成 | 同时提供服务注册发现 | 仅配置管理 |

---

## 第二部分：服务注册与发现（Naming）

### Q9：Nacos 临时实例和持久实例的区别？⭐⭐⭐

**考察点**：两种实例类型的核心差异

**标准答案**：

| 对比项 | 临时实例（ephemeral=true） | 持久实例（ephemeral=false） |
|--------|--------------------------|---------------------------|
| **存储方式** | 纯内存（`ClientManager`） | 磁盘文件（`NamingKvStorage`） |
| **一致性协议** | Distro（AP） | JRaft（CP） |
| **健康检查** | 客户端心跳（被动） | 服务端主动探测（TCP/HTTP/MySQL） |
| **宕机处理** | 心跳超时自动摘除 | 标记为不健康，**不自动删除** |
| **重启恢复** | 不恢复（需重新注册） | 自动恢复（从文件加载） |
| **适用场景** | 微服务（Spring Cloud/Dubbo） | 数据库、中间件等基础设施 |

**⭐ 核心区别**：临时实例"生死与连接绑定"，gRPC 连接断开即视为实例下线；持久实例"独立于连接存在"，需要服务端主动探测才能判断健康状态。

**追问**：默认是临时实例还是持久实例？
> 默认是**临时实例**（`ephemeral=true`），适合绝大多数微服务场景。

---

### Q10：Nacos 2.x 服务注册的完整流程？⭐⭐⭐

**考察点**：gRPC 注册流程、Client 概念

**标准答案**：

```
① 客户端 NamingService.registerInstance()
   → NamingGrpcClientProxy.registerService()
   → 通过 gRPC 长连接发送 InstanceRequest

② 服务端 InstanceRequestHandler.handle()
   → EphemeralClientOperationServiceImpl.registerInstance()

③ 服务端处理：
   a. ClientManager 注册 Client（以 connectionId 为 key）
   b. 创建 Service（若不存在）
   c. 将 Instance 绑定到 Client（Client 持有 publishedService 集合）

④ 发布 ClientRegisterServiceEvent 事件

⑤ NamingSubscriberServiceV2Impl 处理事件：
   a. ServiceStorage 更新服务实例列表（聚合所有 Client 的实例）
   b. 发布 ServiceChangedEvent

⑥ PushDelayTaskExecuteEngine 延迟合并推送任务
   → PushExecuteTask 通过 gRPC 推送给订阅该服务的所有客户端
```

**⭐ 关键设计**：实例与 Client（连接）绑定，gRPC 连接断开时，`ConnectionManager` 触发 `ClientConnectionEventListener`，自动清理该连接关联的所有临时实例。

---

### Q11：Nacos Distro 协议的数据分片是如何实现的？⭐⭐⭐

**考察点**：Distro 哈希算法、数据归属判断

**标准答案**：

**核心类**：`DistroMapper.java`

**哈希算法**（源码）：
```java
// 计算 responsibleTag 归属哪个节点
private int distroHash(String responsibleTag) {
    return Math.abs(responsibleTag.hashCode() % Integer.MAX_VALUE);
}

public boolean responsible(String responsibleTag) {
    final List<String> servers = healthyList;  // 健康节点列表（已排序）
    // ...
    int target = distroHash(responsibleTag) % servers.size();
    return target >= index && target <= lastIndex;  // 判断是否归属当前节点
}
```

**responsibleTag 的含义**：
- **1.x**：`serviceName`（服务名）
- **2.x**：`ip:port`（客户端地址）

**数据归属流程**：
```
客户端注册请求到达节点 A
    │
    ├── DistroMapper.responsible(clientIp:port)
    │     ├── 是：直接处理，异步同步给其他节点
    │     └── 否：DistroFilter 转发给负责节点
```

**节点列表排序**：`healthyList` 在 `onEvent(MembersChangeEvent)` 时通过 `Collections.sort()` 排序，保证所有节点看到的顺序一致，哈希结果才能一致。

**追问**：节点宕机后，原来归属该节点的数据怎么办？
> `healthyList` 更新（移除宕机节点），哈希重新计算，原来的数据由其他节点接管。由于 Distro 是 AP 协议，短暂不一致是允许的，其他节点会通过 `DistroVerifyTimedTask` 定期校验并补全数据。

---

### Q12：Nacos 服务发现（订阅）的流程？⭐⭐

**考察点**：订阅机制、Push 触发

**标准答案**：

```
① 客户端 NamingService.subscribe(serviceName, listener)
   → NamingGrpcClientProxy.subscribe()
   → 发送 SubscribeServiceRequest

② 服务端 SubscribeServiceRequestHandler.handle()
   a. 记录订阅关系（Client -> Service 映射）
   b. 立即返回当前实例列表（快照）

③ 后续服务变更时：
   ServiceChangedEvent → PushDelayTaskExecuteEngine
   → 查询订阅该服务的所有 Client
   → RpcPushService 通过 gRPC 推送 NotifySubscriberRequest

④ 客户端收到推送：
   ServiceInfoHolder 更新本地缓存
   → 触发 InstancesChangeEvent
   → 回调用户注册的 EventListener
```

**客户端本地缓存**：
- 路径：`${user.home}/nacos/naming/{namespace}/`
- 文件名：`{group}@@{serviceName}`
- 服务端不可用时，`FailoverReactor` 从本地缓存加载（故障转移）

---

### Q13：Nacos 保护模式（Protection Mode）是什么？⭐⭐

**考察点**：保护模式触发条件、作用

**标准答案**：

**触发条件**：当健康实例比例低于阈值时触发

```java
// SwitchDomain.java
private float distroThreshold = 0.7F;  // 默认 70%
```

**触发效果**：停止摘除不健康实例（即使心跳超时也不删除）

**设计目的**：防止网络分区时大量实例被误删
- 场景：Nacos 集群与客户端之间网络抖动，大量心跳超时
- 没有保护模式：大量实例被摘除，服务调用失败
- 有保护模式：保留所有实例（包括不健康的），宁可调用失败也不全部摘除

**追问**：保护模式有什么副作用？
> 保护模式下，真正宕机的实例也不会被摘除，消费者可能调用到已宕机的实例，需要客户端有重试机制。

---

### Q14：Nacos 2.x 相比 1.x 有哪些核心改进？⭐⭐⭐

**考察点**：版本演进、架构升级

**标准答案**：

| 改进点 | 1.x | 2.x |
|--------|-----|-----|
| **通信协议** | HTTP + UDP | gRPC 长连接（主）+ HTTP（兼容） |
| **服务端口** | 8848 | 8848（HTTP）+ 9848（gRPC SDK）+ 9849（gRPC 集群） |
| **推送方式** | UDP 推送（不可靠） | gRPC 双向流推送（可靠） |
| **心跳方式** | HTTP 定时心跳（每 5 秒） | gRPC 连接保活（连接断开即下线） |
| **实例管理** | 服务维度 | Client 维度（连接维度） |
| **性能** | 万级实例 | 百万级实例 |
| **推送延迟** | UDP 不可靠，可能丢失 | gRPC 可靠推送，毫秒级 |

**⭐ 最核心的变化**：引入 **Client 概念**（以 connectionId 为 key），实例与连接绑定，连接断开自动清理实例，彻底解决了 1.x 中心跳超时才能感知实例下线的延迟问题。

---

### Q15：Nacos 服务注册时，如果注册请求打到了不负责该实例的节点，会怎样？⭐⭐

**考察点**：Distro 请求转发机制

**标准答案**：

通过 `DistroFilter`（Web 过滤器）处理：

```java
// DistroFilter.java（1.x HTTP 接口）
// 判断当前节点是否负责该请求
if (!distroMapper.responsible(groupedServiceName)) {
    // 不负责 → 转发给负责节点
    String server = distroMapper.mapSrv(groupedServiceName);
    // HTTP 转发到 server
}
```

**2.x gRPC 场景**：
- 客户端连接到哪个节点，该节点就负责处理（Client 与连接绑定）
- 不存在转发问题，因为 Client 的归属就是其连接的节点

**追问**：为什么 2.x 不需要转发？
> 2.x 中 Distro 的分片 key 是 `clientIp:port`（客户端地址），而客户端只连接一个节点，所以该节点天然就是负责节点。

---

### Q16：Nacos 服务实例的心跳超时参数有哪些？⭐⭐

**考察点**：心跳参数配置

**标准答案**：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `heartBeatInterval` | 5 秒 | 客户端发送心跳的间隔 |
| `heartBeatTimeout` | 15 秒 | 超过此时间未收到心跳，标记为**不健康** |
| `ipDeleteTimeout` | 30 秒 | 超过此时间未收到心跳，**删除实例** |

**源码位置**：`com.alibaba.nacos.api.common.Constants.HealthCheckType`

**2.x gRPC 场景**：
- 临时实例不依赖心跳，而是依赖 gRPC 连接保活
- gRPC 连接断开 → 立即触发实例清理（无需等待 15/30 秒）
- 1.x HTTP 心跳仍然兼容支持

---

## 第三部分：一致性协议

### Q17：Nacos 为什么同时使用 CP 和 AP 两种协议？⭐⭐⭐

**考察点**：协议选择策略、CAP 理论应用

**标准答案**：

不同数据对一致性要求不同，Nacos 根据数据类型选择协议：

| 数据类型 | 协议 | 原因 |
|----------|------|------|
| 配置数据 | JRaft（CP） | 配置不能丢失，需要强一致 |
| 持久服务实例 | JRaft（CP） | 持久实例需要强一致，重启后恢复 |
| **临时服务实例** | **Distro（AP）** | 高可用优先，允许短暂不一致，宕机自动摘除 |
| 集群元数据 | JRaft（CP） | 集群状态需要强一致 |

**核心思想**：微服务注册发现场景，**可用性比一致性更重要**。宁可看到短暂不一致的实例列表，也不能因为 Leader 选举期间无法写入而导致服务注册失败。

**追问**：Distro 是 AP 协议，那它如何保证数据不丢失？
> Distro 不保证数据不丢失，但通过以下机制减少数据丢失：① 节点间异步同步；② `DistroVerifyTimedTask` 定期校验；③ 新节点启动时从其他节点全量拉取数据。

---

### Q18：JRaft 的 Raft 选举流程？⭐⭐⭐

**考察点**：Raft 协议基础

**标准答案**：

```
① 初始状态：所有节点为 Follower

② Follower 超时未收到 Leader 心跳（选举超时，默认 5000ms）
   → 转为 Candidate，term++

③ Candidate 向所有节点发送 RequestVote RPC
   - 携带自己的 term 和最后一条日志的 index/term

④ 其他节点投票规则：
   - 本 term 未投过票
   - Candidate 的日志至少和自己一样新（日志完整性检查）
   → 满足条件则投票，并重置选举超时

⑤ Candidate 获得超过半数投票 → 成为 Leader
   - 立即发送心跳（AppendEntries，空日志）通知其他节点

⑥ Leader 定期发送心跳（默认 500ms = 5000/10）维持地位
```

**实测数据**（单节点 standalone 模式）：
```
18:37:06,674 → Node init, term=0
18:37:06,688 → become leader of group, term=1
选举耗时：14ms（单节点直接自选）
```

**追问**：Nacos 有几个 Raft Group？
> **3 个**：`naming_persistent_service_v2`（持久实例）、`naming_service_metadata`（服务元数据）、`naming_instance_metadata`（实例元数据）。每个 Group 独立选举，互不影响。

---

### Q19：JRaft Snapshot 机制是什么？为什么需要它？⭐⭐

**考察点**：Snapshot 作用、触发时机

**标准答案**：

**问题**：Raft 日志会无限增长，新节点加入时需要重放全量日志（可能耗时数小时）。

**解决方案**：Snapshot 将状态机当前状态持久化到磁盘，新节点直接加载 Snapshot，只需重放 Snapshot 之后的少量日志。

**触发时机**：
- 定期触发（默认 1800 秒 = 30 分钟）
- 新节点加入时，Leader 主动发送 Snapshot 给新节点

**存储路径**：
```
${nacos.home}/data/protocol/raft/
├── naming_persistent_service_v2/
│   ├── log/          ← Raft 日志
│   ├── snapshot/snapshot_1/  ← 快照文件（persistent_instance.zip）
│   └── meta-data     ← Raft 元数据（term、votedFor）
```

**实测快照耗时**：3~71ms（取决于数据量）

---

### Q20：Distro 协议的数据同步流程？⭐⭐⭐

**考察点**：Distro 同步机制、延迟合并

**标准答案**：

```
① 节点 A 收到注册请求（归属 A）
   立即写入本地内存

② DistroProtocol.sync(distroKey, action)
   创建 DistroDelayTask（默认延迟 1000ms）

③ DistroDelayTask.merge()：
   同一 key 的多次变更合并为一次（取最新操作类型）
   避免频繁同步

④ 延迟到期，DistroSyncChangeTask 执行
   通过 gRPC 将数据推送给集群其他节点

⑤ 其他节点收到数据，更新本地内存
```

**启动数据加载**：
```
新节点启动 → DistroLoadDataTask
→ 从其他节点拉取全量数据（HTTP GET /distro/datums）
→ 加载完成后才对外提供服务
```

**定期校验**（`DistroVerifyTimedTask`，默认 5 秒）：
- 向其他节点发送自己负责的数据的摘要
- 其他节点比对，发现缺失则请求补全

---

### Q21：Nacos 集群脑裂如何处理？⭐⭐

**考察点**：脑裂问题、Raft 多数派机制

**标准答案**：

**JRaft（CP 协议）防脑裂**：
- Raft 要求超过半数节点确认才能提交日志
- 网络分区时，少数派分区无法选出 Leader（无法获得多数票）
- 少数派分区停止写入，保证数据一致性
- 多数派分区正常工作

**示例**（5 节点集群，网络分区为 3+2）：
- 3 节点分区：可以选出 Leader，正常工作
- 2 节点分区：无法选出 Leader，停止写入（只读）

**Distro（AP 协议）的脑裂处理**：
- Distro 允许脑裂（AP 协议，可用性优先）
- 网络恢复后，通过 `DistroVerifyTimedTask` 数据校验和同步，最终一致

---

## 第四部分：gRPC 通信

### Q22：Nacos 2.x 的 gRPC 通信架构是怎样的？⭐⭐

**考察点**：gRPC 服务端架构、端口规划

**标准答案**：

**服务端组件**：

| 组件 | 端口 | 作用 |
|------|------|------|
| `GrpcSdkServer` | 9848（= 8848 + 1000） | 接受客户端 SDK 连接 |
| `GrpcClusterServer` | 9849（= 8848 + 1001） | 集群节点间通信 |
| `GrpcRequestAcceptor` | - | 处理 Unary RPC 请求 |
| `GrpcBiStreamRequestAcceptor` | - | 处理双向流，维护连接 |
| `ConnectionManager` | - | 管理所有客户端连接 |

**请求处理链**：
```
gRPC 请求
→ GrpcRequestAcceptor
→ RequestFilters（RemoteRequestAuthFilter 鉴权 + TpsControlRequestFilter 限流）
→ RequestHandlerRegistry 查找 Handler
→ 具体 RequestHandler.handle()
```

**追问**：gRPC 端口为什么是 8848 + 1000？
> 硬编码的偏移量设计，简化配置。只需配置 HTTP 端口，gRPC 端口自动计算。防火墙必须同时放开 8848、9848、9849 三个端口。

---

### Q23：Nacos gRPC 连接断开后如何处理？⭐⭐

**考察点**：连接生命周期管理

**标准答案**：

```
gRPC 连接断开
→ GrpcBiStreamRequestAcceptor 检测到流关闭
→ ConnectionManager.unregister(connectionId)
→ 触发 ClientConnectionEventListener.clientDisconnected()
→ ClientManager 清理该连接关联的 Client
→ 发布 ClientDisconnectEvent
→ EphemeralClientOperationServiceImpl 处理事件
→ 删除该 Client 注册的所有临时实例
→ 触发 ServiceChangedEvent
→ Push 通知订阅者（实例列表更新）
```

**僵尸连接清理**（`NacosRuntimeConnectionEjector`）：
- 定期检测连接是否存活（发送 ConnectResetRequest）
- 超时未响应的连接视为僵尸连接，强制断开

---

## 第五部分：健康检查

### Q24：Nacos 健康检查有哪几种方式？⭐⭐

**考察点**：健康检查类型

**标准答案**：

**临时实例**（客户端心跳）：
- 2.x：gRPC 连接保活，连接断开即下线
- 1.x 兼容：HTTP 心跳（`/instance/beat`），默认 5 秒一次

**持久实例**（服务端主动探测）：

| 探测方式 | 实现类 | 适用场景 |
|---------|--------|---------|
| TCP | `TcpHealthCheckProcessor` | 检测端口是否可达 |
| HTTP | `HttpHealthCheckProcessor` | 检测 HTTP 接口响应 |
| MySQL | `MysqlHealthCheckProcessor` | 检测数据库连通性 |
| NONE | - | 不做健康检查，永远健康 |

**追问**：持久实例的健康检查间隔是多少？
> 默认 20 秒（`checkInterval`），可在服务配置中调整。

---

### Q25：Nacos 心跳超时后的处理流程？⭐⭐

**考察点**：心跳检测机制

**标准答案**：

```
ClientBeatCheckTaskV2（定时任务，每 5 秒执行）
    │
    ├── 遍历所有临时实例
    │
    ├── UnhealthyInstanceChecker：
    │   当前时间 - 最后心跳时间 > heartBeatTimeout（15s）
    │   → 标记实例为不健康（healthy=false）
    │   → 发布 ServiceChangedEvent，通知订阅者
    │
    └── ExpiredInstanceChecker：
        当前时间 - 最后心跳时间 > ipDeleteTimeout（30s）
        → 删除实例
        → 发布 ServiceChangedEvent，通知订阅者
```

**拦截器链**（`InstanceBeatCheckTaskInterceptorChain`）：
- 支持自定义拦截器，在健康检查前后插入逻辑
- 内置拦截器：`HealthyCheckInstanceBeatCheckTaskInterceptor`（保护模式检查）

---

## 第六部分：推送机制

### Q26：Nacos 服务变更后如何推送给订阅者？⭐⭐⭐

**考察点**：Push 流程、延迟合并

**标准答案**：

```
服务实例变更（注册/注销/健康状态变化）
    │
    ▼
发布 ServiceChangedEvent
    │
    ▼
NamingSubscriberServiceV2Impl 监听事件
    │
    ▼
PushDelayTaskExecuteEngine（延迟合并，默认 500ms）
    │  同一服务的多次变更合并为一次推送
    ▼
PushExecuteTask 执行
    │
    ├── 查询订阅该服务的所有 Client（订阅关系）
    ├── 构建 NotifySubscriberRequest（包含最新实例列表）
    │
    ▼
RpcPushService.pushWithCallback()
    │
    └── 通过 gRPC 双向流推送给客户端
```

**推送失败重试**：
- 失败后重新加入推送队列
- 最终失败记录到 `PushResultHook`

**追问**：为什么要延迟 500ms 合并推送？
> 服务频繁上下线时（如滚动发布），短时间内可能有大量变更事件。延迟合并可以将多次变更合并为一次推送，减少推送次数，避免推送风暴。

---

### Q27：Nacos 1.x 的 UDP 推送和 2.x 的 gRPC 推送有什么区别？⭐

**考察点**：推送机制演进

**标准答案**：

| 对比项 | 1.x UDP 推送 | 2.x gRPC 推送 |
|--------|-------------|--------------|
| 可靠性 | 不可靠（UDP 可能丢包） | 可靠（TCP 保证） |
| 推送内容 | 仅通知变更，客户端再拉取 | 直接推送完整实例列表 |
| 连接方向 | 服务端主动向客户端 UDP 端口推送 | 复用已有 gRPC 长连接 |
| 防火墙 | 需要客户端开放 UDP 端口 | 无需额外端口 |
| 推送确认 | 无确认机制 | 有 ACK 确认 |

---

## 第七部分：鉴权与安全

### Q28：Nacos 鉴权是如何实现的？⭐⭐

**考察点**：鉴权架构、JWT Token

**标准答案**：

**鉴权架构**（插件化设计）：
```
HTTP 请求 → AuthFilter（core/auth/）
gRPC 请求 → RemoteRequestAuthFilter
    │
    ▼
AuthPluginManager（插件管理器）
    │
    └── NacosAuthPluginService（内置 JWT 实现）
```

**Token 鉴权流程**：
```
① 客户端 POST /v1/auth/users/login（用户名+密码）
② 服务端验证，返回 accessToken（JWT，默认 18000 秒有效期）
③ 后续请求 Header 携带 accessToken
④ AuthFilter 解析 JWT，验证签名和有效期
⑤ 验证通过 → 执行业务逻辑
```

**JWT 密钥**：
- 默认密钥是公开的（`SecretKey012345678901234567890123456789...`）
- **生产必须自定义**：`nacos.core.auth.plugin.nacos.token.secret.key=<Base64编码的自定义密钥>`

---

### Q29：Nacos 生产环境必须做哪些安全加固？⭐⭐

**考察点**：生产安全配置

**标准答案**：

**三大必做项**：

```properties
# ① 开启鉴权（默认关闭！）
nacos.core.auth.enabled=true

# ② 自定义 JWT 密钥（默认密钥是公开的！）
nacos.core.auth.plugin.nacos.token.secret.key=<自定义Base64密钥>

# ③ 关闭 User-Agent 白名单（历史遗留漏洞）
nacos.core.auth.enable.userAgentAuthWhite=false
```

**为什么默认密钥危险**：攻击者可以用公开密钥伪造任意用户的 JWT Token，直接绕过鉴权，获取所有配置和服务信息。

**追加**：集群节点身份认证
```properties
nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=<自定义值>
```

---

## 第八部分：存储层

### Q30：Nacos 配置数据存在哪里？⭐⭐

**考察点**：配置存储架构

**标准答案**：

**三层存储**：
1. **DB**（Derby/MySQL）：持久化，数据不丢失
2. **磁盘文件**（集群模式）：`${nacos.home}/data/config-data/{tenant}/{group}/{dataId}`，供 HTTP 长轮询读取，DB 故障时降级
3. **内存 CacheItem**：只存 MD5 + 元数据，O(1) 判断配置是否变更

**⭐ 特殊情况**：standalone + Derby 模式**不写磁盘**（`PropertyUtil.isDirectRead()=true`），因为 Derby 本身就是持久化的。

---

### Q31：Nacos 服务实例数据存在哪里？⭐⭐

**考察点**：实例存储架构

**标准答案**：

| 实例类型 | 存储位置 | 持久化 | 重启恢复 |
|---------|---------|--------|---------|
| **临时实例** | 纯内存（`ClientManager`） | ❌ | ❌（需重新注册） |
| **持久实例** | `NamingKvStorage`（内存 + 文件双层） | ✅ | ✅ |

**NamingKvStorage 双层设计**：
- 读：优先读内存（快），内存 miss 则读文件并回填内存
- 写：先写文件（持久化），再写内存（缓存）
- 文件路径：`${nacos.home}/data/naming/data/`

**集群模式**：持久实例通过 JRaft 保证集群一致性，`NacosStateMachine.onApply()` 最终调用 `NamingKvStorage.put()`。

---

### Q32：Derby 和 MySQL 如何选择？⭐

**考察点**：数据源选择

**标准答案**：

| 对比项 | Derby（嵌入式） | MySQL（外部） |
|--------|----------------|--------------|
| 适用场景 | 开发测试、单机 | **生产集群（必须）** |
| 集群支持 | 支持（JRaft 保证一致性） | 支持（共享 MySQL） |
| 数据迁移 | 困难 | 容易 |
| 运维复杂度 | 低 | 高（需要维护 MySQL） |
| 性能 | 较低 | 高 |

**生产环境必须使用 MySQL**，Derby 在集群模式下通过 JRaft 同步，性能较差且不便于数据迁移和备份。

---

## 第九部分：生产配置与调优

### Q33：Nacos 集群最少需要几个节点？为什么？⭐⭐

**考察点**：Raft 多数派原理

**标准答案**：

**最少 3 个节点**。

原因：JRaft 使用 Raft 协议，要求超过半数节点存活才能正常工作：
- 3 节点：可容忍 1 个节点故障（2/3 存活）
- 2 节点：任意一个故障即不可用（需要 2/2 存活）

**节点数选择原则**：
- 必须是**奇数**（偶数节点容错能力与少一个奇数节点相同，浪费资源）
- 生产推荐：3 节点（最小高可用）或 5 节点（更高容错）

---

### Q34：Nacos gRPC 线程数是如何计算的？⭐

**考察点**：线程池调优

**标准答案**：

**算法**（`ThreadUtils.getSuitableThreadCount()`）：找第一个 `≥ coreCount × multiple` 的 2 的幂次

```java
public static int getSuitableThreadCount(int threadMultiple) {
    final int coreCount = PropertyUtils.getProcessorsCount();
    int workerCount = 1;
    while (workerCount < coreCount * threadMultiple) {
        workerCount <<= 1;
    }
    return workerCount;
}
```

**gRPC 线程数**（`multiple=16`）：

| CPU 核数 | 计算 | gRPC 线程数 |
|---------|------|------------|
| 4 核 | 4×16=64 | **64** |
| 8 核 | 8×16=128 | **128** |
| 16 核 | 16×16=256 | **256** |

**调整方式**：`-Dremote.executor.times.of.processors=8`（降低 multiple 倍数）

---

### Q35：Nacos 客户端注册失败，如何排查？⭐⭐

**考察点**：生产问题排查

**标准答案**：

**排查步骤**：

1. **检查端口连通性**：
   ```bash
   # 检查 HTTP 端口
   curl http://nacos-host:8848/nacos/v1/console/health/liveness
   # 检查 gRPC 端口（2.x 必须）
   telnet nacos-host 9848
   ```

2. **常见原因**：
   - 防火墙未放开 9848 端口（2.x gRPC 端口）
   - 客户端版本与服务端版本不兼容
   - 鉴权开启但客户端未配置用户名密码
   - 命名空间 ID 配置错误

3. **查看日志**：
   - 服务端：`${nacos.home}/logs/naming-server.log`
   - 客户端：`${user.home}/nacos/logs/naming.log`

4. **检查集群状态**：
   ```bash
   curl http://nacos-host:8848/nacos/v1/core/cluster/nodes
   ```

---

## 第十部分：综合设计题

### Q36：如果让你设计一个配置中心，你会如何设计？⭐⭐⭐

**考察点**：系统设计能力、对 Nacos 的深度理解

**参考答案**（结合 Nacos 设计）：

**核心功能**：
1. 配置存储（DB 持久化）
2. 实时推送（长轮询 or gRPC）
3. 集群一致性（Raft 协议）
4. 客户端容灾（本地快照）

**关键设计决策**：

| 问题 | 设计方案 | 原因 |
|------|---------|------|
| 如何实时推送？ | gRPC 双向流 | 低延迟、可靠、复用连接 |
| 如何保证集群一致？ | Raft 协议 | 强一致，Leader 写入 |
| 如何减少 DB 压力？ | 内存缓存（只存 MD5） | O(1) 判断变更，避免每次查 DB |
| 如何容灾？ | 磁盘文件 + 客户端快照 | 双重保障，DB 故障时降级 |
| 如何支持灰度？ | 按 IP 灰度，独立表存储 | 灵活控制灰度范围 |

---

### Q37：Nacos 和 ZooKeeper 作为注册中心的区别？⭐⭐⭐

**考察点**：技术选型对比

**标准答案**：

| 对比项 | Nacos | ZooKeeper |
|--------|-------|-----------|
| **一致性模型** | AP（临时实例）/ CP（持久实例）可选 | CP（强一致） |
| **健康检查** | 客户端心跳 + 服务端主动探测 | 临时节点（Session 超时自动删除） |
| **推送机制** | gRPC 主动推送 | Watcher 机制 |
| **配置中心** | 内置支持 | 需要额外开发 |
| **控制台** | 内置 Web 控制台 | 无（需要第三方工具） |
| **性能** | 高（百万级实例） | 中（万级节点） |
| **网络分区** | AP 模式下仍可用 | 少数派分区不可用 |

**核心区别**：ZooKeeper 是 CP 系统，网络分区时少数派不可用；Nacos 临时实例是 AP 系统，网络分区时仍可用（但可能看到不一致的数据）。

**面试结论**：微服务场景推荐 Nacos（AP + 功能丰富）；需要强一致的分布式协调场景推荐 ZooKeeper。

---

### Q38：Nacos 如何实现命名空间隔离？⭐

**考察点**：多租户设计

**标准答案**：

- 命名空间（namespace）是 Nacos 的最高隔离层
- 不同命名空间的配置和服务**完全隔离**，互不可见
- 实现方式：所有查询都带 `tenant`（namespace ID）条件
- 默认命名空间：`public`（ID 为空字符串）

**生产建议**：
- 按环境划分：`dev`、`test`、`staging`、`prod`
- 不同环境使用不同命名空间，防止配置污染

---

## 第十一部分：客户端 SDK 内部机制

### Q39：Nacos 客户端断线重连后，已注册的实例会自动恢复吗？如何实现的？⭐⭐⭐

**考察点**：NamingGrpcRedoService 的重试机制

**标准答案**：

**会自动恢复**，由 `NamingGrpcRedoService` 负责。

**核心设计**：`NamingGrpcRedoService` 实现了 `ConnectionEventListener`，监听 gRPC 连接的断开和恢复事件，维护两张重试表：

```
registeredInstances：groupedServiceName → InstanceRedoData（注册重试数据）
subscribes：serviceKey → SubscriberRedoData（订阅重试数据）
```

**`RedoData` 状态机**（4 种 `RedoType`）：

| `registered` | `unregistering` | `RedoType` | 含义 |
|-------------|----------------|-----------|------|
| `false` | `false` | `REGISTER` | 需要重新注册 |
| `true` | `false` | `NONE` | 已注册，无需重试 |
| `true` | `true` | `UNREGISTER` | 需要重新注销 |
| `false` | `true` | `REMOVE` | 注销完成，可删除 |

**完整流程**：

```
① 注册实例时：
   cacheInstanceForRedo()  → registered=false（RedoType=REGISTER）
   注册成功后：instanceRegistered() → registered=true（RedoType=NONE）

② 连接断开时（onDisConnect）：
   connected=false
   所有 registered=true 的实例 → registered=false（RedoType=REGISTER）

③ 连接恢复时（onConnected）：
   connected=true
   RedoScheduledTask 每 3s 扫描：
   → 重新注册所有 RedoType=REGISTER 的实例
   → 重新订阅所有 RedoType=REGISTER 的服务
```

**追问**：如果重连后重新注册失败怎么办？
> `RedoScheduledTask` 每 3 秒执行一次，只要 `connected=true` 且 `RedoType=REGISTER`，就会持续重试，直到成功为止。

---

### Q40：Nacos 客户端的故障转移（FailoverReactor）是如何工作的？⭐⭐

**考察点**：客户端容灾机制

**标准答案**：

`FailoverReactor` 是 Nacos 客户端的**本地磁盘快照容灾机制**，当 Nacos 服务端完全不可用时，从本地磁盘读取服务实例列表，保证业务不中断。

**三个定时任务**：

| 任务 | 间隔 | 作用 |
|------|------|------|
| `DiskFileWriter` | 10 秒 | 将内存中的服务信息写入磁盘快照 |
| `SwitchRefresher` | 5 秒 | 检测故障转移开关文件，决定是否启用故障转移 |
| `DiskFileReader` | 开关打开时触发 | 从磁盘加载所有服务信息到内存 |

**故障转移开关**：在快照目录下创建文件 `00-00---000-VIPSRV_FAILOVER_SWITCH-000---00-00`，内容为 `1` 时开启。

**查询优先级**（`NamingClientProxyDelegate.getServiceInfo()`）：
```
if (failoverReactor.isFailoverSwitch())
    → 从磁盘快照读取（故障转移模式）
else
    → 从内存缓存读取（正常模式）
```

**追问**：故障转移模式下，服务实例列表会更新吗？
> 不会。故障转移模式下，客户端使用的是最后一次写入磁盘的快照，不会再向服务端发起请求。只有关闭故障转移开关后，才恢复正常的实时更新。

---

### Q41：Nacos 客户端如何保证服务信息的最终一致性？⭐⭐

**考察点**：Push + 定时拉取双保险机制

**标准答案**：

Nacos 客户端采用 **"Push 为主 + 定时拉取为辅"** 的双保险机制：

**主动推送（Push）**：
- 服务端实例变更 → 发送 `NotifySubscriberRequest`（gRPC Push）
- 客户端 `NamingPushRequestHandler` 收到后，调用 `ServiceInfoHolder.processServiceInfo()` 更新缓存

**定时拉取（ServiceInfoUpdateService）**：
- 每个订阅的服务都有一个 `UpdateTask`，定时主动拉取
- 自适应间隔：`cacheMillis * failCount`，最大 60 秒
- 作用：**兜底机制**，防止 Push 因网络抖动丢失

**变更检测**：`ServiceInfoHolder.isChangedServiceInfo()` 通过 JSON 序列化比对，只有真正变化时才更新缓存和触发监听器。

**推送空保护**（`pushEmptyProtection=true`）：
- 服务端推送空实例列表时，客户端**保留本地缓存**，不更新
- 防止服务端异常（如 Distro 数据丢失）导致客户端无实例可用

---

### Q42：Nacos 配置客户端（CacheData）如何避免重复触发监听器？⭐⭐

**考察点**：CacheData 的 MD5 比对机制

**标准答案**：

`CacheData` 通过 `lastCallMd5` 字段避免重复触发：

```java
// ManagerListenerWrap 包装用户监听器
private static class ManagerListenerWrap {
    final Listener listener;
    volatile String lastCallMd5;  // 上次触发时的 MD5
}
```

**`checkListenerMd5()` 逻辑**：
```java
for (ManagerListenerWrap wrap : listeners) {
    if (!md5.equals(wrap.lastCallMd5)) {
        // MD5 变化 → 触发监听器（异步）
        safeNotifyListener(..., wrap);
        // 触发后更新 lastCallMd5 = md5
    }
    // MD5 相同 → 跳过，不重复触发
}
```

**关键设计**：
- `md5` 是当前配置内容的 MD5（随配置更新）
- `lastCallMd5` 是上次触发监听器时的 MD5（触发后更新）
- 只有 `md5 ≠ lastCallMd5` 时才触发，保证**同一内容只触发一次**

**异步执行**：监听器回调提交到 `INTERNAL_NOTIFIER` 线程池（最多 5 线程，`SynchronousQueue` 无缓冲），避免阻塞事件总线。

**追问**：如果监听器执行很慢，会影响其他监听器吗？
> 不会。每个监听器独立提交到线程池，互不阻塞。但如果线程池满（5 个线程都在执行），新的通知会被拒绝（`RejectedExecutionException`），该次通知丢失，等下次 MD5 变化时再触发。

---

### Q43：Nacos 配置客户端的 ConfigBatchListenRequest 是什么？⭐⭐

**考察点**：gRPC 配置监听机制

**标准答案**：

`ConfigBatchListenRequest` 是 Nacos 2.x 配置客户端向服务端发送的**批量监听请求**，包含客户端本地所有配置的 `groupKey + MD5`：

```
客户端 → 服务端：ConfigBatchListenRequest
    ├── configListenContexts:
    │     ├── {dataId="app.yaml", group="DEFAULT_GROUP", tenant="dev", md5="abc123"}
    │     ├── {dataId="db.yaml", group="DEFAULT_GROUP", tenant="dev", md5="def456"}
    │     └── ...（所有订阅的配置）

服务端 → 客户端：ConfigChangeBatchListenResponse
    └── changedConfigs: [{dataId="app.yaml", group="DEFAULT_GROUP", tenant="dev"}]
        （只返回 MD5 不一致的配置，即有变更的配置）
```

**触发时机**：
1. 客户端启动时（初始化订阅）
2. 服务端主动 Push `ConfigChangeNotifyRequest`（只含 groupKey，不含内容）
3. 客户端收到 Push 后，重新发送 `ConfigBatchListenRequest`，服务端返回变更列表
4. 客户端再发送 `ConfigQueryRequest` 拉取具体内容

**设计优势**：服务端只需比对 MD5，无需传输配置内容，大幅减少网络传输量。

---

## 第十二部分：TPS 限流与连接控制

### Q44：Nacos 的 TPS 限流是如何实现的？⭐⭐⭐

**考察点**：TPS 限流架构、滑动窗口算法

**标准答案**：

Nacos 通过 `plugin/control/` 模块实现可插拔的 TPS 限流，核心是**环形数组滑动窗口**。

**整体架构**：
```
@TpsControl 注解（标记需要限流的 Handler 方法）
    │
    ▼
TpsControlRequestFilter（gRPC 拦截器）
NacosHttpTpsControlInterceptor（HTTP 拦截器）
    │
    ▼
ControlManagerCenter.getTpsControlManager().check(tpsCheckRequest)
    │
    ▼
NacosTpsControlManager → TpsBarrier → SimpleCountRuleBarrier
    │
    ▼
LocalSimpleCountRateCounter（环形数组滑动窗口）
```

**滑动窗口实现**（`LocalSimpleCountRateCounter`）：
- **10 个槽位**的环形数组，每个槽位对应一个时间窗口
- 槽位定位：`index = (距起始时间的周期数) % 10`
- 槽位过期时自动重置（时间戳不匹配则清零）
- 计数器：`AtomicLong`（无锁，高并发安全）

**两种模式**：
- `MONITOR`：只记录指标，**不拒绝请求**（默认，安全上线）
- `INTERCEPT`：超过阈值时**拒绝请求**，返回 HTTP 503 / gRPC OVER_THRESHOLD

**动态热更新**：规则变更通过 `TpsControlRuleChangeEvent` 事件驱动，`TpsBarrier.applyRule()` 热更新，无需重启。

**追问**：如何自定义限流算法（如令牌桶）？
> 实现 `RuleBarrierCreator` SPI 接口，在 `createRuleBarrier()` 中返回自定义的 `RuleBarrier`（内部使用令牌桶），在 `META-INF/services/` 中注册，并通过 `nacos.plugin.control.tps.barrier.creator` 配置指定名称。

---

### Q45：Nacos TPS 限流的规则如何持久化和动态更新？⭐⭐

**考察点**：规则存储与热更新机制

**标准答案**：

**规则持久化路径**：`${nacos.home}/data/tps/{pointName}`（本地磁盘）

**规则更新流程**：
```
管理员通过 API 更新规则
    │
    ▼
ControlManagerCenter.reloadTpsControlRule(pointName, external)
    │
    ▼
NotifyCenter.publishEvent(TpsControlRuleChangeEvent)
    │
    ▼
ControlRuleChangeActivator.TpsRuleChangeSubscriber.onEvent()
    │
    ├── external=true：从外部存储拉取规则，写入本地磁盘
    ├── 从本地磁盘读取规则内容
    ├── RuleParser 解析为 TpsControlRule 对象
    │
    ▼
TpsControlManager.applyTpsRule(pointName, rule)
    └── TpsBarrier.applyRule(rule)  // 热更新，无需重启
```

**外部存储扩展**：实现 `ExternalRuleStorage` SPI，可将规则存储到 Nacos 配置中心或数据库，实现集群级别的规则统一管理。

**TPS 指标上报**：`NacosTpsControlManager` 内置定时任务（每 900ms），将各限流点的通过数/拒绝数写入 `tps.log`：
```
{pointName}|point|{period}|{时间}|{passCount}|{deniedCount}
```

---

### Q46：Nacos 连接数控制是如何实现的？默认行为是什么？⭐

**考察点**：连接控制机制

**标准答案**：

**连接控制规则**（`ConnectionControlRule`）：
```java
private int countLimit = -1;          // 最大连接数（-1 表示不限）
private Set<String> monitorIpList;    // 需要监控的 IP 列表
```

**默认行为**：`NacosConnectionControlManager`（默认实现）的 `check()` 方法**直接返回 CHECK_SKIP**（跳过检查），即**默认不限制连接数**。这是有意为之的设计，避免误伤。

**连接数统计**：`ConnectionMetricsReporter` 每 3 秒将各来源的连接数写入 `connection.log`：
```
ConnectionMetrics, totalCount = 1024, detail = {sdk=980, cluster=44}
```

**规则持久化路径**：`${nacos.home}/data/connection/limitRule`

**追问**：如何开启连接数限制？
> 通过 API 更新 `ConnectionControlRule`，设置 `countLimit > 0`，并确保 `NacosConnectionControlManager` 的 `check()` 方法实现了实际的限制逻辑（默认实现直接跳过，需要自定义实现或使用企业版）。

---

## 第十三部分：能力协商机制

### Q47：Nacos 2.x 的能力协商机制是什么？解决了什么问题？⭐⭐

**考察点**：Ability 协商机制、版本兼容性

**标准答案**：

**能力协商**是 Nacos 2.x 在 gRPC 连接建立时引入的**版本兼容机制**。

**解决的问题**：客户端/服务端版本不一致时的兼容性问题。例如：
- 老客户端（不支持增量推送）连接新服务端 → 服务端不发增量推送，发全量
- 新客户端连接老服务端（不支持某功能）→ 客户端降级处理

**数据结构**：
```
ClientAbilities（客户端上报）
    ├── ClientRemoteAbility（supportRemoteConnection）
    ├── ClientConfigAbility（supportRemoteMetrics）
    └── ClientNamingAbility（supportDeltaPush、supportRemoteMetric）

ServerAbilities（服务端返回）
    ├── ServerRemoteAbility
    ├── ServerConfigAbility
    └── ServerNamingAbility
```

**交换时机**：gRPC 双向流建立阶段（`ConnectionSetupRequest`）：
```
客户端 → 服务端：ConnectionSetupRequest（含 ClientAbilities）
服务端：GrpcBiStreamRequestAcceptor 解析，存入 Connection 对象
服务端处理请求时：connection.getAbilities().getNamingAbility().isSupportDeltaPush()
                  → 决定发增量还是全量
```

**服务端能力初始化**：通过 SPI（`ServerAbilityInitializer`）扩展，各模块独立注册，符合开闭原则。

**追问**：如何新增一个能力字段？
> ① 在 `ClientNamingAbility`/`ServerNamingAbility` 中添加字段；② 实现 `ServerAbilityInitializer` 设置服务端能力；③ 在 `META-INF/services/` 中注册；④ 服务端处理请求时通过 `connection.getAbilities()` 判断走哪条路径。

---

### Q48：Nacos 集群节点间如何感知彼此的能力？⭐

**考察点**：集群能力同步

**标准答案**：

`ServerAbilities` 不仅用于客户端协商，也用于**集群节点间的能力感知**：

- 每个 `Member`（集群节点）对象携带 `ServerAbilities`
- 节点间通过心跳（`/cluster/report`）交换 `Member` 信息，包含各节点的 `ServerAbilities`
- 集群中每个节点都知道其他节点支持哪些能力，从而在集群内部通信时选择最优协议

**实际应用**：集群内部 gRPC 通信（`GrpcClusterServer`，端口 9849）时，发送方可以根据目标节点的 `ServerAbilities` 决定使用哪种请求格式，保证集群内部的版本兼容性。

---

## 第十四部分：任务引擎体系

### Q49：Nacos 的 NacosTaskExecuteEngine 体系是什么？为什么需要两种引擎？⭐⭐⭐

**考察点**：任务引擎设计、延迟合并原理

**标准答案**：

Nacos 的任务引擎体系是一个通用的**异步任务调度框架**，有两种实现：

| 引擎 | 任务类型 | 存储 | 调度方式 | 合并能力 |
|------|---------|------|---------|---------|
| `NacosDelayTaskExecuteEngine` | `AbstractDelayTask` | `ConcurrentHashMap<key, task>` | 单线程定时扫描（100ms） | ✅ 同 key 合并 |
| `NacosExecuteTaskExecuteEngine` | `AbstractExecuteTask` | `TaskExecuteWorker[]` | 多 Worker 并行 | ❌ 不合并 |

**Naming 推送中的双引擎协作**：

```
ServiceChangedEvent
    │
    ▼
PushDelayTaskExecuteEngine（NacosDelayTaskExecuteEngine）
    │ 延迟 500ms，合并同一 Service 的多次变更
    │ 到期后：PushDelayTaskProcessor.process(task)
    ▼
NamingExecuteTaskDispatcher（NacosExecuteTaskExecuteEngine）
    │ 按 service.hashCode() 分配到对应 Worker
    │ Worker 线程执行 PushExecuteTask.run()
    ▼
gRPC Push → 客户端
```

**为什么需要两个引擎**：
- **变更收集阶段**：需要合并短时间内的多次变更（用延迟合并引擎）
- **推送执行阶段**：每个 Service 的推送需要立即执行，且不同 Service 可以并行（用立即执行引擎）

**追问**：任务合并的具体逻辑是什么？
> `addTask(key, newTask)` 时，若已有同 key 的旧任务，调用 `newTask.merge(existTask)`（新任务吸收旧任务的信息），然后用新任务替换旧任务。对于 `PushDelayTask`，合并后取最新的 revision，延迟时间重置为 500ms。

---

### Q50：NacosDelayTaskExecuteEngine 的任务到期判断逻辑是什么？⭐

**考察点**：延迟任务的调度机制

**标准答案**：

**`shouldProcess()` 判断**（`AbstractDelayTask`）：
```java
public boolean shouldProcess() {
    // 当前时间 - 最后处理时间 >= 任务间隔（taskInterval）
    return (System.currentTimeMillis() - this.lastProcessTime >= this.taskInterval);
}
```

**`ProcessRunnable` 每 100ms 扫描一次**：
```
遍历所有任务 key
    │
    ├── task.shouldProcess() = false → 重新放回队列（未到期）
    │
    └── task.shouldProcess() = true → 查找 NacosTaskProcessor 执行
                                       执行失败 → 重新放回队列（重试）
```

**`PushDelayTask` 的参数**：
- `taskInterval = 500ms`（`PushConfig.getPushTaskDelay()`）
- `lastProcessTime = System.currentTimeMillis()`（创建时设置）
- 因此：任务创建 500ms 后，`shouldProcess()` 返回 `true`，触发执行

**任务引擎在其他场景的应用**：

| 场景 | 引擎类型 | 合并策略 |
|------|---------|---------|
| Naming 推送（变更收集） | `NacosDelayTaskExecuteEngine` | 同 Service 合并，取最新状态 |
| Distro 数据同步 | `NacosDelayTaskExecuteEngine` | 同 key 合并，取最新 action |
| Config Dump | `NacosDelayTaskExecuteEngine` | 同 groupKey 合并，只 dump 一次 |
| Naming 推送（实际执行） | `NacosExecuteTaskExecuteEngine` | 不合并，立即执行 |

---

## 总结：面试高频考点速查

### 必背知识点

| 知识点 | 关键数字/结论 |
|--------|-------------|
| 长轮询超时 | 29.5 秒（30000 - 500ms） |
| 心跳间隔 | 5 秒 |
| 心跳超时（不健康） | 15 秒 |
| 心跳超时（删除） | 30 秒 |
| 保护模式阈值 | 70%（`distroThreshold=0.7F`） |
| JRaft 选举超时 | 5000ms |
| JRaft 心跳间隔 | 500ms（= 5000/10） |
| JRaft Snapshot 间隔 | 1800 秒（30 分钟） |
| Distro 同步延迟 | 1000ms |
| Distro 校验间隔 | 5000ms |
| gRPC 端口偏移 | +1000（SDK）、+1001（集群） |
| JRaft Group 数量 | 3 个 |
| 推送延迟合并 | 500ms |
| gRPC 线程数（8核） | 128（= 8×16，取2的幂） |
| RpcClient 心跳间隔 | 5000ms（`connectionKeepAlive`） |
| RpcClient 心跳重试 | 3 次（`healthCheckRetryTimes`） |
| RpcClient 心跳超时 | 3000ms（`healthCheckTimeOut`） |
| NamingGrpcRedoService 重试间隔 | 3000ms（`DEFAULT_REDO_DELAY`） |
| FailoverReactor 磁盘写入间隔 | 10 秒 |
| FailoverReactor 开关检测间隔 | 5 秒 |
| ServiceInfoUpdateService 最大刷新间隔 | 60 秒 |
| TPS 滑动窗口槽位数 | 10 个 |
| TPS 指标上报间隔 | 900ms |
| 连接数指标上报间隔 | 3 秒 |
| 任务引擎扫描间隔 | 100ms（`processInterval`） |
| PushDelayTask 延迟 | 500ms（`DEFAULT_PUSH_TASK_DELAY`） |

### 高频面试题 TOP 15

1. **配置实时推送原理**（长轮询 vs gRPC）→ Q1
2. **临时实例 vs 持久实例**（AP vs CP）→ Q9
3. **服务注册完整流程**（gRPC + Client 概念）→ Q10
4. **Distro 数据分片算法**（哈希 + 排序节点列表）→ Q11
5. **为什么同时用 CP 和 AP**（数据类型决定协议）→ Q17
6. **Raft 选举流程**（term + 多数派）→ Q18
7. **Nacos 2.x vs 1.x 核心改进**（gRPC + Client）→ Q14
8. **保护模式**（70% 阈值 + 防误删）→ Q13
9. **生产安全加固**（三大必做项）→ Q29
10. **Nacos vs ZooKeeper**（AP vs CP + 功能对比）→ Q37
11. **断线重连后实例自动恢复**（NamingGrpcRedoService）→ Q39
12. **TPS 限流实现原理**（环形数组滑动窗口）→ Q44
13. **能力协商机制**（ClientAbilities + ServerAbilities）→ Q47
14. **任务引擎双引擎协作**（延迟合并 + 立即执行）→ Q49
15. **CacheData 避免重复触发监听器**（lastCallMd5）→ Q42

---

*文档生成时间：2026-03-05*  
*对应源码版本：Nacos 2.x*  
*覆盖章节：第1章～第15章全部核心知识点（含客户端SDK、TPS限流、能力协商、任务引擎）*
