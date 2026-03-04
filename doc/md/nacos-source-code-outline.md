# Nacos 源码学习大纲

> 版本：Nacos 2.x  
> 目标：生产环境核心原理 + 高频面试考点  
> 模块路径参考：`/data/workspace/nacos`

---

## 目录

1. [Nacos 整体架构概览](#1-nacos-整体架构概览)
2. [启动流程与模块初始化](#2-启动流程与模块初始化)
3. [配置中心（Config）核心原理](#3-配置中心config核心原理)
4. [服务注册与发现（Naming）核心原理](#4-服务注册与发现naming核心原理)
5. [一致性协议：CP（JRaft）与 AP（Distro）](#5-一致性协议cpjraft与apdistro)
6. [客户端通信机制：gRPC 长连接](#6-客户端通信机制grpc-长连接)
7. [集群管理与节点发现](#7-集群管理与节点发现)
8. [健康检查机制](#8-健康检查机制)
9. [推送机制（Push）](#9-推送机制push)
10. [鉴权与安全](#10-鉴权与安全)
11. [存储层设计](#11-存储层设计)
12. [生产环境核心配置与调优](#12-生产环境核心配置与调优)
13. [高频面试题汇总](#13-高频面试题汇总)

---

## 1. Nacos 整体架构概览

### 1.1 核心功能定位

| 功能 | 说明 | 对应模块 |
|------|------|----------|
| 配置管理 | 动态配置下发、灰度发布 | `config/` |
| 服务注册发现 | 服务注册、健康检查、服务订阅 | `naming/` |
| 一致性协议 | CP（JRaft）/ AP（Distro） | `core/distributed/` |
| 客户端 SDK | Java/Go/Python 客户端 | `client/` |
| 控制台 | Web 管理界面 | `console/` + `console-ui/` |

### 1.2 模块依赖关系

```
nacos-console (入口)
    ├── nacos-config      (配置中心)
    ├── nacos-naming      (服务注册发现)
    ├── nacos-core        (核心基础设施)
    │     ├── distributed/raft    (JRaft CP协议)
    │     ├── distributed/distro  (Distro AP协议)
    │     ├── remote/grpc         (gRPC通信)
    │     └── cluster             (集群管理)
    ├── nacos-consistency  (一致性协议抽象层)
    ├── nacos-auth         (鉴权)
    └── nacos-client       (客户端SDK)
```

### 1.3 Nacos 2.x vs 1.x 核心变化

| 对比项 | 1.x | 2.x |
|--------|-----|-----|
| 通信协议 | HTTP + UDP | gRPC 长连接（主）+ HTTP（兼容） |
| 服务端口 | 8848 | 8848（HTTP）+ 9848（gRPC）+ 9849（gRPC集群） |
| 推送方式 | UDP 推送 | gRPC 双向流推送 |
| 心跳方式 | HTTP 定时心跳 | gRPC 连接保活 |
| 性能 | 较低 | 大幅提升（连接数、TPS） |

---

## 2. 启动流程与模块初始化

### 2.1 启动入口

- 入口类：`NamingApp.java` / `Config.java`（各模块独立启动）
- 统一入口：`nacos-console` 模块的 Spring Boot 主类
- 关键监听器：`StartingApplicationListener.java`（`core/listener/`）

### 2.2 启动流程时序

```
SpringBoot 启动
    │
    ├── StartingApplicationListener.onStarting()
    │       ├── 打印 Banner
    │       ├── 设置运行模式（standalone/cluster）
    │       └── 初始化系统属性
    │
    ├── ServerMemberManager 初始化（集群成员管理）
    │       ├── LookupFactory 选择成员发现方式
    │       │     ├── StandaloneMemberLookup（单机）
    │       │     ├── FileConfigMemberLookup（cluster.conf）
    │       │     └── AddressServerMemberLookup（地址服务器）
    │       └── 启动成员健康检测任务
    │
    ├── ProtocolManager 初始化（一致性协议）
    │       ├── JRaftProtocol（CP，持久化数据）
    │       └── DistroProtocol（AP，临时数据）
    │
    ├── gRPC Server 启动
    │       ├── GrpcSdkServer（端口 9848，客户端连接）
    │       └── GrpcClusterServer（端口 9849，集群内部通信）
    │
    ├── Config 模块初始化
    │       ├── DumpService 启动（全量/增量 dump）
    │       └── LongPollingService 初始化
    │
    └── Naming 模块初始化
            ├── ServiceManager 初始化
            └── HealthCheckReactor 启动
```

### 2.3 关键源码位置

| 类名 | 路径 | 作用 |
|------|------|------|
| `StartingApplicationListener` | `core/listener/` | 启动监听，初始化环境 |
| `ServerMemberManager` | `core/cluster/` | 集群成员管理核心类 |
| `ProtocolManager` | `core/distributed/` | 一致性协议管理器 |
| `LookupFactory` | `core/cluster/lookup/` | 成员发现策略工厂 |
| `DumpService` | `config/service/dump/` | 配置数据 dump 服务 |

---

## 3. 配置中心（Config）核心原理

### 3.1 配置数据模型

```
ConfigInfo
    ├── dataId      (配置ID)
    ├── group       (分组，默认 DEFAULT_GROUP)
    ├── tenant      (命名空间/租户)
    ├── content     (配置内容)
    └── md5         (内容MD5，用于变更检测)

GroupKey = dataId + "+" + group + "+" + tenant
```

### 3.2 配置发布流程

```
客户端 publishConfig()
    │
    ▼
ConfigController.publishConfig() / ConfigPublishRequestHandler（gRPC）
    │
    ▼
ConfigOperationService.publishConfig()
    │
    ├── 写入数据库（config_info 表）
    ├── 发布 ConfigDataChangeEvent 事件
    │
    ▼
AsyncNotifyService（异步通知）
    │
    ├── 通知集群其他节点（HTTP/gRPC）
    │
    ▼
DumpService.dump()
    │
    ├── 写入本地磁盘缓存（/nacos/data/config-data/）
    ├── 更新内存 CacheItem（MD5）
    │
    ▼
LongPollingService / RpcConfigChangeNotifier
    │
    └── 通知订阅该配置的客户端
```

### 3.3 长轮询（Long Polling）机制 ⭐⭐⭐

**核心类**：`LongPollingService.java`（`config/service/`）

**原理**：
1. 客户端发起 HTTP 请求，携带本地配置的 MD5 列表
2. 服务端比对 MD5，若有变更立即返回变更的 dataId 列表
3. 若无变更，服务端**挂起请求**（默认 29.5 秒），加入 `allSubs` 队列
4. 配置变更时，触发 `LocalDataChangeEvent`，遍历 `allSubs` 找到订阅该配置的客户端，立即响应
5. 超时后返回空响应，客户端重新发起长轮询

```java
// 关键代码路径
LongPollingService
    ├── addLongPollingClient()   // 挂起客户端请求
    ├── DataChangeTask           // 配置变更时触发推送
    └── ClientLongPolling        // 封装挂起的客户端请求（含超时任务）
```

**面试要点**：
- 为什么是 29.5 秒而不是 30 秒？客户端默认发送 `Long-Pulling-Timeout: 30000`（ms），服务端计算 `timeout = 30000 - 500ms`（`delayTime=500`），提前 500ms 响应，避免客户端超时（源码：`LongPollingService.addLongPollingClient()`）
- 长轮询 vs 短轮询 vs WebSocket 的对比
- 2.x 中 gRPC 如何替代长轮询？（`RpcConfigChangeNotifier` 主动推送）

### 3.4 配置 Dump 机制 ⭐⭐

**核心类**：`DumpService.java`（`config/service/dump/`）
- `ExternalDumpService`：外部数据库（MySQL）模式
- `EmbeddedDumpService`：嵌入式数据库（Derby）模式

**作用**：将数据库中的配置同步到本地磁盘和内存缓存

**触发时机**：
- 启动时全量 dump（`DumpAllTask`）
- 配置变更时增量 dump（`DumpTask`）
- 定时全量 dump（每 6 小时）

**本地缓存路径**：`${nacos.home}/data/config-data/{tenant}/{group}/{dataId}`

**意义**：
- 数据库故障时，服务端仍可从磁盘提供配置（降级保障）
- 减少数据库查询压力

### 3.5 集群配置同步

```
Leader 节点收到配置变更
    │
    ▼
AsyncNotifyService 异步通知 Follower 节点
    │
    ├── HTTP: /v1/cs/communication/dataChange
    └── gRPC: ConfigChangeClusterSyncRequest
    │
    ▼
Follower 节点执行 dump，更新本地缓存
```

### 3.6 灰度发布（Beta 配置）

- 支持按 IP 灰度：`config_info_beta` 表
- 客户端请求时携带 IP，服务端判断是否命中灰度规则
- 关键类：`ConfigController.publishConfigBeta()`

---

## 4. 服务注册与发现（Naming）核心原理

### 4.1 服务数据模型

```
Service（服务）
    ├── namespace
    ├── group
    ├── serviceName
    └── Cluster[]
            └── Instance[]
                    ├── ip
                    ├── port
                    ├── weight
                    ├── healthy
                    ├── ephemeral  (临时/持久实例)
                    └── metadata
```

### 4.2 临时实例 vs 持久实例 ⭐⭐⭐

| 对比项 | 临时实例（ephemeral=true） | 持久实例（ephemeral=false） |
|--------|--------------------------|---------------------------|
| 存储方式 | 内存（Distro AP协议） | 磁盘（JRaft CP协议） |
| 健康检查 | 客户端心跳 | 服务端主动探测（TCP/HTTP/MySQL） |
| 宕机处理 | 心跳超时自动摘除 | 标记为不健康，不自动删除 |
| 适用场景 | 微服务（Spring Cloud/Dubbo） | 数据库、中间件等基础设施 |
| 一致性模型 | AP（高可用优先） | CP（强一致优先） |

### 4.3 服务注册流程 ⭐⭐⭐

**2.x gRPC 注册流程**：

```
客户端 registerInstance()
    │
    ▼
NamingGrpcClientProxy.registerService()
    │  (gRPC 长连接)
    ▼
InstanceRequestHandler（服务端）
    │
    ▼
EphemeralClientOperationServiceImpl.registerInstance()
    │
    ├── ClientManager 注册 Client（连接维度）
    ├── 创建 Service（若不存在）
    ├── 将 Instance 绑定到 Client
    │
    ▼
发布 ClientRegisterServiceEvent
    │
    ▼
NamingSubscriberServiceV2Impl 处理事件
    │
    ├── ServiceStorage 更新服务实例列表
    └── 触发 Push 通知订阅者
```

**核心类**：
- `InstanceRequestHandler`（`naming/remote/rpc/`）
- `EphemeralClientOperationServiceImpl`（`naming/core/v2/service/`）
- `ClientManager`（`naming/core/v2/client/`）

### 4.4 服务发现（订阅）流程

```
客户端 subscribe(serviceName, listener)
    │
    ▼
NamingGrpcClientProxy.subscribe()
    │
    ▼
SubscribeServiceRequestHandler（服务端）
    │
    ├── 记录订阅关系（Client -> Service 映射）
    ├── 立即返回当前实例列表
    │
    ▼
服务变更时，Push 通知客户端（gRPC 双向流）
    │
    ▼
客户端 ServiceInfoHolder 更新本地缓存
    └── 触发 InstancesChangeEvent，回调 EventListener
```

### 4.5 Distro 协议在 Naming 中的应用 ⭐⭐

**核心思想**：每个节点只负责一部分数据（按 IP 哈希分片），节点间异步同步

```
客户端注册请求到达节点 A
    │
    ├── 判断该 Instance 是否归属节点 A（DistroMapper）
    │     ├── 是：直接处理，异步同步给其他节点
    │     └── 否：转发给负责节点（DistroFilter）
    │
    ▼
DistroProtocol.sync()
    │
    └── 异步将数据同步给集群其他节点
```

**关键类**：
- `DistroMapper`（`naming/core/`）：计算实例归属节点
- `DistroFilter`（`naming/web/`）：请求转发过滤器
- `DistroProtocol`（`core/distributed/distro/`）：Distro 协议实现

---

## 5. 一致性协议：CP（JRaft）与 AP（Distro）

### 5.1 协议选择策略

| 数据类型 | 协议 | 原因 |
|----------|------|------|
| 配置数据 | JRaft（CP） | 配置需要强一致，不能丢失 |
| 持久服务实例 | JRaft（CP） | 持久实例需要强一致 |
| 临时服务实例 | Distro（AP） | 高可用优先，允许短暂不一致 |
| 集群元数据 | JRaft（CP） | 集群状态需要强一致 |

### 5.2 JRaft（CP 协议）⭐⭐⭐

**基于 SOFAJRaft 实现**，Raft 共识算法

**核心流程**：
```
写请求 → Leader 节点
    │
    ├── 写入 Raft Log
    ├── 并行复制给 Follower（超过半数确认）
    ├── Leader 提交（Apply）
    │
    ▼
NacosStateMachine.onApply()
    │
    └── 执行实际的业务逻辑（写数据库/内存）
```

**关键类**：
- `JRaftServer`（`core/distributed/raft/`）：JRaft 服务器封装
- `JRaftProtocol`（`core/distributed/raft/`）：CP 协议实现
- `NacosStateMachine`（`core/distributed/raft/`）：状态机，处理日志应用
- `RaftConfig`（`core/distributed/raft/`）：Raft 配置参数

**Snapshot 机制**：
- 定期将状态机快照到磁盘（`JSnapshotOperation`）
- 新节点加入时通过 Snapshot 快速同步，避免重放全量日志

### 5.3 Distro（AP 协议）⭐⭐⭐

**自研协议**，专为临时服务实例设计

**核心设计**：
1. **数据分片**：每个节点负责一部分数据（按客户端 IP 哈希）
2. **最终一致**：节点间异步同步，允许短暂不一致
3. **无中心化**：无 Leader，所有节点对等

**数据同步流程**：
```
节点 A 收到注册请求（归属 A）
    │
    ├── 立即写入本地内存
    │
    ▼
DistroProtocol.sync()
    │
    ├── 延迟任务（DistroDelayTask）合并同一 key 的多次变更
    │
    ▼
DistroSyncChangeTask 执行
    │
    └── 通过 gRPC 将数据推送给其他节点
```

**启动数据加载**：
```
新节点启动
    │
    ▼
DistroLoadDataTask
    │
    ├── 从其他节点拉取全量数据（DistroDataStorage）
    └── 加载完成后才对外提供服务
```

**关键类**：
- `DistroProtocol`（`core/distributed/distro/`）：Distro 协议核心
- `DistroDelayTask`（`core/distributed/distro/task/delay/`）：延迟合并任务
- `DistroVerifyTimedTask`（`core/distributed/distro/task/verify/`）：定期校验数据一致性

---

## 6. 客户端通信机制：gRPC 长连接

### 6.1 架构设计

```
客户端                          服务端
    │                              │
    │  gRPC 双向流（BiStream）      │
    │ ─────────────────────────── ▶│  GrpcBiStreamRequestAcceptor
    │                              │
    │  Unary RPC（请求/响应）       │
    │ ─────────────────────────── ▶│  GrpcRequestAcceptor
    │                              │
    │◀ ─────────────────────────── │  服务端主动 Push（配置变更/实例变更）
```

### 6.2 服务端 gRPC 组件

| 类名 | 端口 | 作用 |
|------|------|------|
| `GrpcSdkServer` | 9848 | 接受客户端 SDK 连接 |
| `GrpcClusterServer` | 9849 | 集群节点间通信 |
| `GrpcRequestAcceptor` | - | 处理 Unary 请求 |
| `GrpcBiStreamRequestAcceptor` | - | 处理双向流，维护连接 |
| `ConnectionManager` | - | 管理所有客户端连接 |

### 6.3 连接管理

**核心类**：`ConnectionManager.java`（`core/remote/`）

```
客户端建立 gRPC 连接
    │
    ▼
GrpcBiStreamRequestAcceptor 接受连接
    │
    ├── 创建 GrpcConnection 对象
    ├── 注册到 ConnectionManager
    │
    ▼
ConnectionManager
    ├── 维护 connectionId -> Connection 映射
    ├── 定期检测僵尸连接（NacosRuntimeConnectionEjector）
    └── 连接断开时触发 ClientConnectionEventListener
```

### 6.4 请求处理链

```
gRPC 请求到达
    │
    ▼
GrpcRequestAcceptor.request()
    │
    ▼
RequestFilters（过滤器链）
    ├── RemoteRequestAuthFilter（鉴权）
    └── TpsControlRequestFilter（限流）
    │
    ▼
RequestHandlerRegistry 查找对应 RequestHandler
    │
    ▼
具体 RequestHandler.handle()（如 InstanceRequestHandler）
```

### 6.5 客户端重连机制

- 客户端维护 gRPC 连接，断线后自动重连
- 重连时重新注册服务实例、重新订阅配置
- 服务端连接断开时，自动清理该连接关联的临时实例

---

## 7. 集群管理与节点发现

### 7.1 成员发现方式

**核心类**：`LookupFactory.java`（`core/cluster/lookup/`）

| 方式 | 类名 | 适用场景 |
|------|------|----------|
| 单机模式 | `StandaloneMemberLookup` | 开发测试 |
| 配置文件 | `FileConfigMemberLookup` | 固定 IP 集群 |
| 地址服务器 | `AddressServerMemberLookup` | 动态扩缩容 |

### 7.2 集群成员管理

**核心类**：`ServerMemberManager.java`（`core/cluster/`）

```
ServerMemberManager
    ├── 维护集群成员列表（Member[]）
    ├── 定期向其他节点发送心跳（/cluster/report）
    ├── 更新节点状态（UP/DOWN/SUSPICIOUS）
    └── 发布 MembersChangeEvent（通知 Raft/Distro 等组件）
```

### 7.3 节点状态

```java
// core/src/main/java/com/alibaba/nacos/core/cluster/NodeState.java（源码实际有5种状态）
enum NodeState {
    STARTING,    // 节点正在启动，不对外提供服务
    UP,          // 正常，可处理请求
    SUSPICIOUS,  // 可疑（心跳超时但未确认下线）
    DOWN,        // 下线，停止服务
    ISOLATION,   // 被隔离（人工干预）
}
```

> ⚠️ 注意：大纲早期版本写的是"3种状态"，源码实际有 **5 种**，已通过读取 `NodeState.java` 源码验证。

---

## 8. 健康检查机制

### 8.1 临时实例健康检查（客户端心跳）

**2.x 机制**：gRPC 连接保活（连接断开即视为实例下线）

**1.x 兼容机制**：HTTP 心跳

```
客户端定时发送心跳（默认 5 秒）
    │
    ▼
InstanceController.beat() / ClientBeatProcessorV2
    │
    ├── 更新实例最后心跳时间
    │
    ▼
ClientBeatCheckTaskV2（定时任务）
    │
    ├── 通过 InstanceBeatCheckTaskInterceptorChain 执行拦截器链
    ├── UnhealthyInstanceChecker：心跳超时（默认 15 秒）→ 标记为不健康
    └── ExpiredInstanceChecker：删除超时（默认 30 秒）→ 删除实例
```

**关键超时参数**：
- `heartBeatInterval`：心跳间隔，默认 5 秒
- `heartBeatTimeout`：心跳超时，默认 15 秒（标记不健康）
- `ipDeleteTimeout`：删除超时，默认 30 秒（删除实例）

### 8.2 持久实例健康检查（服务端主动探测）

**支持的探测方式**：
- **TCP**：建立 TCP 连接检测端口是否可达
- **HTTP**：发送 HTTP 请求检测响应状态码
- **MySQL**：执行 SQL 检测数据库连通性

**核心类**：
- `HealthCheckTaskV2`（`naming/healthcheck/v2/`）
- `TcpHealthCheckProcessor`（`naming/healthcheck/v2/processor/`）
- `HttpHealthCheckProcessor`（`naming/healthcheck/v2/processor/`）
- `MysqlHealthCheckProcessor`（`naming/healthcheck/v2/processor/`）

### 8.3 保护模式（Protection Mode）⭐⭐

**触发条件**：当健康实例比例低于阈值（默认 **70%**）时触发

**效果**：停止摘除不健康实例，防止网络分区时大量实例被误删

```java
// SwitchDomain.java
private float distroThreshold = 0.7F;  // Distro 保护阈值，默认 70%
```

---

## 9. 推送机制（Push）

### 9.1 2.x gRPC Push 流程 ⭐⭐

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
PushDelayTaskExecuteEngine（延迟合并，避免频繁推送）
    │
    ▼
PushExecuteTask
    │
    ├── 查询订阅该服务的所有客户端
    ├── 构建 NotifySubscriberRequest
    │
    ▼
RpcPushService.pushWithCallback()
    │
    └── 通过 gRPC 双向流推送给客户端
```

**关键类**：
- `RpcPushService`（`core/remote/`）：gRPC 推送服务
- `PushDelayTaskExecuteEngine`（`naming/push/v2/`）：推送延迟任务引擎

### 9.2 推送重试机制

- 推送失败后进行重试（默认重试 3 次）
- 重试间隔指数退避
- 最终失败记录到 `PushResultHook`

### 9.3 1.x UDP Push（了解即可）

- 服务端通过 UDP 向客户端推送变更通知
- 客户端收到 UDP 通知后，主动发起 HTTP 请求拉取最新数据
- 2.x 中已被 gRPC 双向流替代

---

## 10. 鉴权与安全

### 10.1 鉴权架构

```
HTTP 请求 → AuthFilter（core/auth/）
gRPC 请求 → RemoteRequestAuthFilter（core/auth/）
    │
    ▼
AuthPluginManager（插件化鉴权）
    │
    ├── 内置实现：NacosAuthPluginService（JWT Token）
    └── 自定义扩展：实现 AuthPluginService 接口
```

### 10.2 Token 鉴权流程

```
客户端登录（/v1/auth/login）
    │
    ▼
返回 accessToken（JWT，默认 18000 秒有效期）
    │
    ▼
后续请求携带 accessToken
    │
    ▼
AuthFilter 验证 Token 有效性
    │
    └── 验证通过 → 执行业务逻辑
```

### 10.3 命名空间隔离

- 不同命名空间（namespace）的配置和服务完全隔离
- 生产环境建议按环境（dev/test/prod）划分命名空间

---

## 11. 存储层设计

### 11.1 配置存储

| 存储方式 | 适用场景 | 实现类 |
|----------|----------|--------|
| 嵌入式 Derby | 单机/集群（内嵌） | `LocalDataSourceServiceImpl` |
| 外部 MySQL | 生产集群（推荐） | `ExternalDataSourceServiceImpl` |

**关键表**：
- `config_info`：配置主表
- `config_info_beta`：灰度配置
- `config_info_tag`：标签配置
- `his_config_info`：配置历史记录

### 11.2 服务实例存储

- **临时实例**：纯内存存储（`ClientManager`），不持久化
- **持久实例**：通过 JRaft 写入 RocksDB（嵌入式 KV 存储）

### 11.3 KV 存储

**核心类**：`KvStorage`（`core/storage/kv/`）

| 实现 | 说明 |
|------|------|
| `MemoryKvStorage` | 内存 KV，用于临时数据 |
| `FileKvStorage` | 文件 KV，用于持久化 |

---

## 12. 生产环境核心配置与调优

### 12.1 集群部署建议

```properties
# cluster.conf 配置集群节点
192.168.1.1:8848
192.168.1.2:8848
192.168.1.3:8848

# 推荐奇数节点（3/5/7），满足 Raft 多数派要求
```

### 12.2 数据库配置（MySQL）

```properties
# application.properties
spring.datasource.platform=mysql
db.num=2  # 主从双数据库
db.url.0=jdbc:mysql://master:3306/nacos?...
db.url.1=jdbc:mysql://slave:3306/nacos?...
db.user=nacos
db.password=nacos
```

### 12.3 JVM 调优

```bash
# 推荐配置（8C16G 机器）
-Xms4g -Xmx4g -Xmn2g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
```

### 12.4 关键配置参数

```properties
# 开启鉴权
nacos.core.auth.enabled=true
nacos.core.auth.plugin.nacos.token.secret.key=<自定义密钥>

# 连接数限制
nacos.remote.server.rpc.tls.enable=false

# 配置 dump 间隔（秒）
dump.change.worker.count=10

# 服务端健康检查线程数
healthCheckProcessorThreadCount=1
```

### 12.5 监控指标

Nacos 集成 Prometheus + Micrometer，关键指标：

| 指标 | 说明 |
|------|------|
| `nacos_monitor_service_count` | 服务数量 |
| `nacos_monitor_ip_count` | 实例数量 |
| `nacos_monitor_subscriber_count` | 订阅者数量 |
| `nacos_monitor_long_polling` | 长轮询连接数 |
| `nacos_grpc_connection_count` | gRPC 连接数 |

---

## 13. 高频面试题汇总

### 13.1 配置中心相关

**Q1：Nacos 配置中心如何实现配置的实时推送？**

> 1.x：长轮询（Long Polling）。客户端发起 HTTP 请求，服务端挂起 29.5 秒，配置变更时立即响应。
> 2.x：gRPC 双向流主动推送。服务端配置变更后，通过 `RpcConfigChangeNotifier` 主动推送给订阅的客户端，延迟更低。

**Q2：Nacos 配置中心的 Dump 机制是什么？有什么作用？**

> Dump 机制将数据库中的配置同步到本地磁盘和内存缓存（CacheItem + MD5）。
> 作用：① 减少数据库查询压力；② 数据库故障时提供降级保障；③ 长轮询通过比对内存中的 MD5 快速判断配置是否变更。

**Q3：Nacos 集群中配置如何同步？**

> Leader 节点收到配置变更后，通过 `AsyncNotifyService` 异步通知所有 Follower 节点执行 dump，更新本地缓存。配置数据通过 JRaft 保证强一致性。

### 13.2 服务注册发现相关

**Q4：Nacos 临时实例和持久实例的区别？**

> 见 4.2 节对比表。核心区别：临时实例用 AP（Distro），客户端心跳维活，宕机自动摘除；持久实例用 CP（JRaft），服务端主动探测，宕机标记不健康但不删除。

**Q5：Nacos 服务注册的流程是什么？**

> 见 4.3 节。2.x 通过 gRPC 长连接注册，服务端将实例绑定到 Client（连接维度），连接断开时自动清理临时实例。

**Q6：Nacos 的 Distro 协议是如何工作的？**

> Distro 是 AP 协议，核心思想是数据分片：每个节点只负责一部分数据（按客户端 IP 哈希分配），节点间异步同步。写请求若不归属当前节点，则转发给负责节点。节点启动时从其他节点拉取全量数据。

**Q7：Nacos 保护模式是什么？**

> 当健康实例比例低于阈值（默认 **70%**，`SwitchDomain.distroThreshold = 0.7F`）时，Nacos 停止摘除不健康实例，防止网络分区时大量实例被误删，保护服务可用性。该阈值可通过控制台或 API 动态调整。

### 13.3 一致性协议相关

**Q8：Nacos 为什么同时使用 CP 和 AP 两种协议？**

> 不同数据对一致性要求不同：配置数据和持久实例需要强一致（CP/JRaft），临时实例需要高可用（AP/Distro）。Nacos 通过 `ProtocolManager` 统一管理两种协议，根据数据类型选择对应协议。

**Q9：JRaft 的 Raft 选举流程？**

> ① Follower 超时未收到 Leader 心跳，转为 Candidate；② Candidate 向所有节点发送 RequestVote；③ 获得超过半数投票后成为 Leader；④ Leader 定期发送心跳维持地位。

**Q10：Nacos 2.x 相比 1.x 有哪些核心改进？**

> ① 通信协议从 HTTP+UDP 升级为 gRPC 长连接，性能大幅提升；② 服务端主动 Push 替代长轮询，延迟更低；③ 引入 Client 概念（连接维度），连接断开自动清理实例；④ 支持更大规模的服务实例（百万级）。

### 13.4 生产问题排查

**Q11：Nacos 集群脑裂如何处理？**

> JRaft 通过 Raft 多数派机制避免脑裂：只有获得超过半数节点确认的 Leader 才能提交日志。网络分区时，少数派分区无法选出 Leader，停止写入，保证数据一致性。

**Q12：Nacos 客户端配置拉取失败如何降级？**

> 客户端本地有快照缓存（`${user.home}/nacos/config/`），服务端不可用时从本地快照加载配置，保证服务不中断。

---

## 源码阅读路径建议

### 第一阶段：基础架构（1-2天）
1. 阅读 `StartingApplicationListener` 了解启动流程
2. 阅读 `ServerMemberManager` 了解集群管理
3. 阅读 `BaseGrpcServer` / `GrpcRequestAcceptor` 了解 gRPC 通信

### 第二阶段：配置中心（2-3天）
1. `ConfigController` → `ConfigOperationService` → 数据库写入
2. `LongPollingService` 长轮询核心逻辑
3. `DumpService` 配置 dump 机制
4. `RpcConfigChangeNotifier` gRPC 推送

### 第三阶段：服务注册发现（2-3天）
1. `InstanceRequestHandler` → `EphemeralClientOperationServiceImpl`
2. `DistroProtocol` → `DistroDelayTask` → `DistroSyncChangeTask`
3. `ClientBeatCheckTaskV2` 心跳检测
4. `RpcPushService` 推送机制

### 第四阶段：一致性协议（2-3天）
1. `JRaftServer` → `NacosStateMachine` JRaft 核心
2. `DistroProtocol` Distro 协议完整流程
3. `ProtocolManager` 协议管理器

---

*文档生成时间：2026-03-04*  
*对应源码版本：Nacos 2.x*
