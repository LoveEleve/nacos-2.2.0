# Nacos 整体架构概览

> 本文件是理解 Nacos 源码的**第一步**，建议在阅读任何具体模块前先通读本文。  
> 对应源码版本：Nacos 2.x | 阅读导航：[00-reading-guide.md](./00-reading-guide.md)

---

## 目录

1. [整体架构分层](#1-整体架构分层)
2. [核心模块依赖关系](#2-核心模块依赖关系)
3. [全链路请求追踪](#3-全链路请求追踪)
   - 3.1 [服务实例注册全链路](#31-服务实例注册全链路)
   - 3.2 [服务订阅与推送全链路](#32-服务订阅与推送全链路)
   - 3.3 [配置发布全链路](#33-配置发布全链路)
4. [各模块职责边界](#4-各模块职责边界)
5. [关键设计决策](#5-关键设计决策)
6. [2.x vs 1.x 核心差异](#6-2x-vs-1x-核心差异)

---

## 1. 整体架构分层

Nacos 2.x 整体分为 **四层**：

```
┌─────────────────────────────────────────────────────────────────┐
│                        客户端 SDK 层                              │
│  NacosNamingService  │  NacosConfigService  │  NacosFactory      │
├─────────────────────────────────────────────────────────────────┤
│                        通信协议层                                  │
│  gRPC（2.x 主协议）  │  HTTP（1.x 兼容）  │  能力协商机制          │
├─────────────────────────────────────────────────────────────────┤
│                        服务端核心层                                │
│  注册中心（Naming）  │  配置中心（Config）  │  集群管理（Cluster）  │
├─────────────────────────────────────────────────────────────────┤
│                        基础设施层                                  │
│  一致性协议（JRaft/Distro）│ 存储（Derby/MySQL）│ 流量控制（TPS）  │
└─────────────────────────────────────────────────────────────────┘
```

### 端口规划

| 端口 | 协议 | 用途 | 对应类 |
|------|------|------|--------|
| `8848` | HTTP | 控制台 + 1.x API | `NacosApplicationListener` |
| `9848` | gRPC | 客户端 SDK 连接 | `GrpcSdkServer` |
| `9849` | gRPC | 集群节点间通信 | `GrpcClusterServer` |

> **规律**：gRPC 端口 = HTTP 端口 + 偏移量（SDK +1000，集群 +1001）

---

## 2. 核心模块依赖关系

### 2.1 整体模块依赖图

```mermaid
graph TB
    subgraph CLIENT["客户端 SDK"]
        NNS[NacosNamingService\n服务注册发现门面]
        NCS[NacosConfigService\n配置中心门面]
        RPC[RpcClient\ngRPC连接管理]
        CW[ClientWorker\n配置监听管理]
        SIH[ServiceInfoHolder\n服务信息缓存]
        FR[FailoverReactor\n故障转移]
        NGRS[NamingGrpcRedoService\n断线重试]
    end

    subgraph COMM["通信层"]
        NGCP[NamingGrpcClientProxy\n客户端代理]
        GBS[GrpcBiStreamRequestAcceptor\n双向流接入]
        GRA[GrpcRequestAcceptor\nUnary请求接入]
        CM[ConnectionManager\n连接管理]
        AN[AbilityNegotiation\n能力协商]
    end

    subgraph NAMING["注册中心服务端"]
        IH[InstanceRequestHandler\n实例注册处理]
        SSH[SubscribeServiceRequestHandler\n订阅处理]
        CLM[ClientManager\nClient管理]
        CSIM[ClientServiceIndexesManager\n双向索引]
        SS[ServiceStorage\n服务实例聚合]
        PM[PushDelayTaskExecuteEngine\n推送引擎]
    end

    subgraph CONFIG["配置中心服务端"]
        CPH[ConfigPublishRequestHandler\n配置发布]
        CQH[ConfigQueryRequestHandler\n配置查询]
        CCS[ConfigCacheService\n配置内存缓存]
        CIPS[ConfigInfoPersistService\n配置持久化]
        DS[DumpService\nDump机制]
        RCCN[RpcConfigChangeNotifier\n配置推送]
    end

    subgraph CONSISTENCY["一致性协议"]
        DP[DistroProtocol\nAP协议]
        JP[JRaftProtocol\nCP协议]
        DCDP[DistroClientDataProcessor\nDistro数据处理]
        NSM[NacosStateMachine\nRaft状态机]
    end

    subgraph CLUSTER["集群管理"]
        SMM[ServerMemberManager\n成员管理]
        LF[LookupFactory\n寻址策略]
        ASMLL[AddressServerMemberLookup\n地址服务器寻址]
        FCML[FileConfigMemberLookup\n文件寻址]
    end

    subgraph INFRA["基础设施"]
        TCM[TpsControlManager\nTPS限流]
        CCM[ConnectionControlManager\n连接控制]
        SL[StorageLayer\n存储层]
        HC[HealthCheckReactor\n健康检查]
    end

    NNS --> NGCP --> RPC
    NCS --> CW --> RPC
    NNS --> SIH
    NNS --> FR
    RPC --> NGRS

    RPC -->|gRPC连接| GBS
    RPC -->|gRPC请求| GRA
    GBS --> CM
    GBS --> AN
    GRA --> IH
    GRA --> SSH
    GRA --> CPH
    GRA --> CQH

    IH --> CLM
    IH --> CSIM
    SSH --> CSIM
    CSIM --> SS
    SS --> PM
    PM -->|推送| CM

    CPH --> CCS
    CPH --> CIPS
    DS --> CCS
    RCCN --> CM

    IH -->|临时实例| DP
    IH -->|持久实例| JP
    DP --> DCDP
    JP --> NSM

    SMM --> LF
    LF --> ASMLL
    LF --> FCML

    CM --> TCM
    CM --> CCM
    CIPS --> SL
    CLM --> HC
```

### 2.2 数据流向总览

```mermaid
flowchart LR
    C[客户端] -->|gRPC 9848| S[服务端]
    S -->|gRPC Push| C

    S1[节点A] -->|gRPC 9849\nDistro同步| S2[节点B]
    S1 -->|gRPC 9849\nRaft日志| S3[节点C]

    S -->|写| DB[(Derby/MySQL)]
    S -->|读| DISK[磁盘文件\n/nacos/data/]
    S -->|内存| MEM[内存缓存\nConcurrentHashMap]
```

---

## 3. 全链路请求追踪

### 3.1 服务实例注册全链路

> 场景：Spring Boot 应用启动，向 Nacos 注册一个**临时实例**

```mermaid
sequenceDiagram
    participant APP as 应用程序
    participant NNS as NacosNamingService
    participant NGCP as NamingGrpcClientProxy
    participant RC as RpcClient
    participant GRA as GrpcRequestAcceptor
    participant IRH as InstanceRequestHandler
    participant TCM as TpsControlManager
    participant CLM as ClientManager
    participant CSIM as ClientServiceIndexesManager
    participant SS as ServiceStorage
    participant DP as DistroProtocol
    participant PM as PushDelayTaskExecuteEngine

    APP->>NNS: registerInstance(serviceName, instance)
    NNS->>NGCP: registerService(...)
    NGCP->>RC: request(InstanceRequest)

    Note over RC: 检查连接状态\n若未连接则先建立gRPC连接

    RC->>GRA: gRPC Unary 请求 (端口9848)
    GRA->>TCM: passRateLimit(InstanceRequest)

    alt TPS 超限
        TCM-->>GRA: 拒绝
        GRA-->>RC: TpsFlowControlResponse(FAIL)
    else TPS 正常
        GRA->>IRH: handle(InstanceRequest, requestMeta)
        IRH->>CLM: getClient(connectionId)
        IRH->>CSIM: addInstanceToService(clientId, namespace, service, instance)
        CSIM->>SS: 更新服务实例聚合数据
        CSIM->>DP: publishEvent(ClientChangedEvent)

        Note over DP: 异步同步到其他节点\n(DistroDelayTask 合并)

        SS->>PM: addTask(PushDelayTask, 500ms延迟)

        Note over PM: 500ms后触发\n查询所有订阅者并推送

        IRH-->>GRA: InstanceResponse(SUCCESS)
        GRA-->>RC: gRPC响应
        RC-->>NGCP: InstanceResponse
        NGCP-->>NNS: 注册成功
        NNS-->>APP: 返回
    end
```

**关键路径说明：**

| 步骤 | 核心类 | 关键操作 |
|------|--------|---------|
| ① 客户端发起 | `NamingGrpcClientProxy` | 封装 `InstanceRequest`，调用 `RpcClient.request()` |
| ② TPS 检查 | `TpsControlRequestFilter` | 滑动窗口计数，超限返回 `TpsFlowControlResponse` |
| ③ 处理注册 | `InstanceRequestHandler` | 区分 `REGISTER`/`DEREGISTER` 操作类型 |
| ④ 更新索引 | `ClientServiceIndexesManager` | 维护 `clientId→services` 和 `service→clients` 双向 Map |
| ⑤ 触发推送 | `PushDelayTaskExecuteEngine` | 500ms 延迟合并，避免频繁推送 |
| ⑥ Distro 同步 | `DistroProtocol` | 异步同步到其他节点（AP 最终一致） |

---

### 3.2 服务订阅与推送全链路

> 场景：消费者订阅服务，服务提供者上线后消费者收到推送

```mermaid
sequenceDiagram
    participant CONSUMER as 消费者应用
    participant NNS as NacosNamingService
    participant SIUS as ServiceInfoUpdateService
    participant SIH as ServiceInfoHolder
    participant NGCP as NamingGrpcClientProxy
    participant SSRH as SubscribeServiceRequestHandler
    participant CSIM as ClientServiceIndexesManager
    participant PROVIDER as 提供者应用
    participant SS as ServiceStorage
    participant PDTE as PushDelayTaskExecuteEngine
    participant RPS as RpcPushService
    participant CM as ConnectionManager

    Note over CONSUMER: 订阅阶段
    CONSUMER->>NNS: subscribe(serviceName, listener)
    NNS->>NGCP: subscribe(serviceName)
    NGCP->>SSRH: SubscribeServiceRequest
    SSRH->>CSIM: addSubscriberToService(clientId, service)
    SSRH-->>NGCP: SubscribeServiceResponse(当前实例列表)
    NGCP->>SIH: processServiceInfo(serviceInfo)
    SIH->>SIH: 写入内存缓存 + 磁盘快照
    SIH-->>NNS: 触发 InstancesChangeEvent
    NNS-->>CONSUMER: 回调 listener.onEvent()

    Note over CONSUMER: 定时刷新（兜底）
    SIUS->>NGCP: 定时 queryInstancesOfService()
    NGCP-->>SIH: 更新缓存（若MD5变化则触发事件）

    Note over PROVIDER: 提供者上线
    PROVIDER->>SS: 注册新实例（同上注册流程）
    SS->>PDTE: addTask(PushDelayTask, 500ms)

    Note over PDTE: 延迟合并执行
    PDTE->>PDTE: 500ms 后触发 PushExecuteTask
    PDTE->>CSIM: 查询 service 的所有订阅者 clientIds
    PDTE->>SS: getInstancesOfService() 获取最新实例列表
    loop 每个订阅者
        PDTE->>CM: getConnection(clientId)
        PDTE->>RPS: pushWithCallback(connection, NotifySubscriberRequest)
        RPS-->>CONSUMER: gRPC Push（NotifySubscriberRequest）
        CONSUMER->>SIH: processServiceInfo(serviceInfo)
        SIH-->>CONSUMER: 触发 listener.onEvent()
    end
```

**推送失败重试机制：**

```
RpcPushService.pushWithCallback()
    ├── 第1次推送失败
    │   └── 延迟 1s 重试（最多3次）
    ├── 第2次推送失败
    │   └── 延迟 3s 重试
    └── 第3次推送失败
        └── 放弃，等待下次 ServiceInfoUpdateService 定时拉取兜底
```

---

### 3.3 配置发布全链路

> 场景：运维人员通过控制台修改配置，客户端实时感知变更

```mermaid
sequenceDiagram
    participant OPS as 运维人员/控制台
    participant CPH as ConfigPublishRequestHandler
    participant CCS as ConfigCacheService
    participant CIPS as ConfigInfoPersistService
    participant DB as 数据库(Derby/MySQL)
    participant DS as DumpService
    participant DISK as 磁盘文件
    participant ANS as AsyncNotifyService
    participant OTHER as 其他集群节点
    participant RCCN as RpcConfigChangeNotifier
    participant CM as ConnectionManager
    participant CW as ClientWorker
    participant CD as CacheData
    participant APP as 应用程序

    Note over OPS: 发布配置
    OPS->>CPH: ConfigPublishRequest(dataId, group, content)
    CPH->>CIPS: insertOrUpdate(configInfo)
    CIPS->>DB: INSERT/UPDATE config_info
    DB-->>CIPS: 成功
    CIPS->>DS: 触发 ConfigDataChangeEvent

    Note over DS: Dump 流程（DB→磁盘→内存）
    DS->>DB: 查询最新配置
    DS->>DISK: 写入 /nacos/data/config-data/{tenant}/{group}/{dataId}
    DS->>CCS: 更新内存缓存（MD5）
    DS->>RCCN: 触发 LocalDataChangeEvent

    Note over ANS: 集群同步
    DS->>ANS: 通知其他节点同步
    ANS->>OTHER: ConfigChangeClusterSyncRequest
    OTHER->>OTHER: 执行本地 Dump 流程

    Note over RCCN: 推送客户端
    RCCN->>CM: 查询监听该配置的所有连接
    loop 每个监听客户端
        RCCN->>CM: getConnection(clientId)
        RCCN-->>CW: ConfigChangeNotifyRequest（gRPC Push）
    end

    Note over CW: 客户端处理推送
    CW->>CW: 收到 ConfigChangeNotifyRequest
    CW->>CPH: 主动拉取 ConfigQueryRequest
    CPH->>CCS: 查询内存缓存（命中则直接返回）
    CCS-->>CW: 返回最新配置内容
    CW->>CD: 更新 CacheData（content + MD5）
    CD->>CD: 比对 MD5，若变化则触发监听器
    CD-->>APP: 回调 Listener.receiveConfigInfo()
```

**三级查询策略（客户端查询配置）：**

```
NacosConfigService.getConfig()
    ├── 第1级：FailoverReactor（故障转移文件）
    │   └── 若 failover-mode 开关文件存在 → 直接读磁盘快照返回
    ├── 第2级：远程服务端（正常情况）
    │   └── ConfigQueryRequest → ConfigQueryRequestHandler
    │       ├── 优先读 ConfigCacheService 内存缓存
    │       └── 缓存未命中 → 读数据库
    └── 第3级：本地快照（服务端不可用时）
        └── 读 /nacos/data/config-data/ 磁盘文件
```

---

## 4. 各模块职责边界

### 4.1 客户端 SDK 模块

```mermaid
graph LR
    subgraph SDK["客户端 SDK（nacos-client）"]
        direction TB
        A[NacosNamingService\n门面类，对外API] --> B[NamingGrpcClientProxy\ngRPC代理，封装请求]
        A --> C[ServiceInfoHolder\n本地缓存+磁盘快照]
        A --> D[FailoverReactor\n故障转移，读磁盘]
        B --> E[RpcClient\n连接状态机+重连]
        E --> F[NamingGrpcRedoService\n断线重试队列]
        C --> G[ServiceInfoUpdateService\n定时拉取兜底]
    end
```

| 类 | 职责 | 边界 |
|----|------|------|
| `NacosNamingService` | 对外暴露 API（注册/注销/订阅/查询） | **不**直接操作网络，委托给 Proxy |
| `NamingGrpcClientProxy` | 封装 gRPC 请求，处理响应 | **不**管理连接生命周期，委托给 RpcClient |
| `RpcClient` | 管理 gRPC 连接（状态机：STARTING→RUNNING→UNHEALTHY→SHUTDOWN） | **不**关心业务逻辑，只负责连接 |
| `NamingGrpcRedoService` | 维护待重试操作队列（注册/订阅） | **只**在连接恢复时触发，不主动发起连接 |
| `ServiceInfoHolder` | 内存缓存 + 磁盘快照（`/nacos/naming/`） | **不**主动拉取，被动接收推送和定时刷新结果 |
| `FailoverReactor` | 读取故障转移磁盘文件，开关文件控制 | **只**在 failover 模式下生效，正常情况不介入 |

---

### 4.2 通信层模块

```mermaid
graph TB
    subgraph SERVER_COMM["服务端通信层（nacos-core）"]
        GBS[GrpcBiStreamRequestAcceptor\n双向流接入\n处理连接建立+能力协商]
        GRA[GrpcRequestAcceptor\nUnary请求接入\n分发到RequestHandler]
        CM[ConnectionManager\n连接注册表\n健康检测+僵尸清理]
        RHR[RequestHandlerRegistry\nHandler注册表\nSpring Bean自动注册]
        TPS[TpsControlRequestFilter\nTPS限流过滤器\n滑动窗口计数]
        AUTH[RemoteRequestAuthFilter\n鉴权过滤器\nJWT验证]
    end

    GBS --> CM
    GRA --> TPS --> AUTH --> RHR
```

| 类 | 职责 | 边界 |
|----|------|------|
| `GrpcBiStreamRequestAcceptor` | 接受双向流，注册连接，处理能力协商 | **不**处理业务请求，只管连接生命周期 |
| `GrpcRequestAcceptor` | 接受 Unary 请求，经过过滤器链后分发 | **不**直接处理业务，委托给 RequestHandler |
| `ConnectionManager` | 维护 `connectionId→Connection` 注册表，定期检测僵尸连接 | **不**关心连接上的业务数据，只管连接本身 |
| `RequestHandlerRegistry` | Spring 启动时扫描所有 `RequestHandler` Bean 并注册 | **只**负责注册和查找，不执行处理逻辑 |
| `TpsControlRequestFilter` | 每个请求类型独立的 TPS 屏障（`NacosTpsBarrier`） | **只**做计数和限流，不修改请求内容 |

---

### 4.3 注册中心服务端模块

```mermaid
graph TB
    subgraph NAMING_SERVER["注册中心服务端（nacos-naming）"]
        IRH[InstanceRequestHandler\n实例注册/注销]
        SSRH[SubscribeServiceRequestHandler\n服务订阅]
        CLM[ClientManager\n管理所有Client]
        CSIM[ClientServiceIndexesManager\n双向索引维护]
        SS[ServiceStorage\n服务实例聚合]
        PDTE[PushDelayTaskExecuteEngine\n推送延迟引擎]
        RPS[RpcPushService\ngRPC推送服务]
        HC[HealthCheckReactor\n持久实例健康检查]
    end

    IRH --> CLM
    IRH --> CSIM
    SSRH --> CSIM
    CSIM --> SS
    SS --> PDTE
    PDTE --> RPS
    CLM --> HC
```

| 类 | 职责 | 边界 |
|----|------|------|
| `InstanceRequestHandler` | 处理注册/注销请求，区分临时/持久实例走不同路径 | **不**直接操作存储，委托给 `EphemeralClientOperationServiceImpl` |
| `ClientManager` | 管理 `clientId→Client` 映射，监听连接断开事件自动清理 | **不**关心 Client 内部的服务数据 |
| `ClientServiceIndexesManager` | 维护双向索引：`clientId→Set<Service>` 和 `Service→Set<clientId>` | **只**维护索引，不存储实例数据 |
| `ServiceStorage` | 聚合所有 Client 的实例数据，提供 `getInstancesOfService()` | **不**直接推送，发布事件触发推送引擎 |
| `PushDelayTaskExecuteEngine` | 500ms 延迟合并推送任务，避免频繁推送 | **不**直接发送 gRPC，委托给 `RpcPushService` |

---

### 4.4 配置中心服务端模块

```mermaid
graph TB
    subgraph CONFIG_SERVER["配置中心服务端（nacos-config）"]
        CPH[ConfigPublishRequestHandler\n配置发布]
        CQH[ConfigQueryRequestHandler\n配置查询]
        CCS[ConfigCacheService\n内存缓存（MD5+路径）]
        CIPS[ConfigInfoPersistService\n数据库读写]
        DS[DumpService\nDB→磁盘→内存]
        ANS[AsyncNotifyService\n集群同步通知]
        RCCN[RpcConfigChangeNotifier\n推送客户端]
        LPS[LongPollingService\n1.x长轮询兼容]
    end

    CPH --> CIPS --> DS
    DS --> CCS
    DS --> ANS
    DS --> RCCN
    CQH --> CCS
```

| 类 | 职责 | 边界 |
|----|------|------|
| `ConfigCacheService` | 内存缓存（`groupKey→MD5+文件路径`），提供快速查询 | **不**直接读数据库，只读内存和磁盘文件 |
| `ConfigInfoPersistService` | 数据库 CRUD，支持 Derby（嵌入式）和 MySQL | **不**更新内存缓存，写完 DB 后触发 Dump |
| `DumpService` | 启动时全量 Dump，运行时增量 Dump（DB→磁盘→内存） | **不**直接推送客户端，发布 `LocalDataChangeEvent` |
| `AsyncNotifyService` | 异步通知集群其他节点执行 Dump | **只**负责集群内同步，不推送客户端 |
| `RpcConfigChangeNotifier` | 监听 `LocalDataChangeEvent`，推送给监听该配置的客户端 | **只**发送通知（`ConfigChangeNotifyRequest`），不发送配置内容 |

---

### 4.5 一致性协议模块

```mermaid
graph LR
    subgraph CONSISTENCY["一致性协议（nacos-consistency）"]
        PM[ProtocolManager\n协议管理器]
        JP[JRaftProtocol\nCP协议]
        DP[DistroProtocol\nAP协议]
        JRS[JRaftServer\nMultiRaftGroup]
        NSM[NacosStateMachine\nonApply执行业务]
        DCDP[DistroClientDataProcessor\n数据处理器]
        DCTA[DistroClientTransportAgent\n集群传输代理]
    end

    PM --> JP --> JRS --> NSM
    PM --> DP --> DCDP
    DP --> DCTA
```

| 类 | 职责 | 边界 |
|----|------|------|
| `ProtocolManager` | 统一管理 JRaft 和 Distro 两种协议 | **不**直接处理数据，委托给具体协议实现 |
| `JRaftProtocol` | CP 协议，用于**持久实例**和**配置数据** | **只**处理需要强一致性的数据 |
| `DistroProtocol` | AP 协议，用于**临时实例**数据 | **只**处理临时实例，允许短暂不一致 |
| `NacosStateMachine` | Raft 日志应用，`onApply()` 执行实际业务操作 | **不**直接操作网络，通过 `RequestHandler` 执行 |
| `DistroClientDataProcessor` | 处理 Distro 数据的序列化/反序列化和同步 | **只**处理 Client 维度的数据（临时实例） |

---

### 4.6 集群管理模块

```mermaid
graph TB
    subgraph CLUSTER["集群管理（nacos-core）"]
        SMM[ServerMemberManager\n成员注册表+状态管理]
        LF[LookupFactory\n寻址策略工厂]
        ASMLL[AddressServerMemberLookup\n地址服务器寻址]
        FCML[FileConfigMemberLookup\n文件寻址]
        SMLL[SingleMemberLookup\n单机模式]
        MIRT[MemberInfoReportTask\n节点心跳上报]
        CVJ[ClusterVersionJudgement\n版本判断]
    end

    LF --> ASMLL
    LF --> FCML
    LF --> SMLL
    SMM --> MIRT
    SMM --> CVJ
```

| 类 | 职责 | 边界 |
|----|------|------|
| `ServerMemberManager` | 维护集群成员列表，管理节点状态（UP/SUSPICIOUS/DOWN） | **不**直接发现节点，委托给 `MemberLookup` |
| `LookupFactory` | 根据配置选择寻址策略（SPI 扩展点） | **只**负责创建策略，不执行寻址 |
| `AddressServerMemberLookup` | 定期从地址服务器拉取成员列表（生产环境推荐） | **只**负责发现，不管理节点状态 |
| `FileConfigMemberLookup` | 监听 `cluster.conf` 文件变化（inotify 热更新） | **只**负责发现，不管理节点状态 |
| `ClusterVersionJudgement` | 判断集群是否全部升级到 2.x，控制兼容性开关 | **只**做版本判断，不执行升级操作 |

---

## 5. 关键设计决策

### 5.1 为什么同时使用 JRaft（CP）和 Distro（AP）？

```
临时实例（Ephemeral）
  ├── 特点：随客户端连接存在，连接断开自动删除
  ├── 一致性要求：最终一致即可（允许短暂不一致）
  ├── 性能要求：高吞吐（大量服务频繁注册/注销）
  └── 选择：Distro（AP）✅

持久实例（Persistent）
  ├── 特点：独立于客户端连接，需要主动注销
  ├── 一致性要求：强一致（不能丢失）
  ├── 性能要求：相对较低（变更不频繁）
  └── 选择：JRaft（CP）✅

配置数据
  ├── 特点：全局共享，变更需要立即生效
  ├── 一致性要求：强一致（配置错误影响全局）
  └── 选择：JRaft（CP）✅
```

### 5.2 为什么 gRPC 推送后还需要定时拉取兜底？

```
gRPC 推送（主动）
  ├── 优点：实时性好（毫秒级）
  ├── 缺点：推送失败无法保证送达（网络抖动、客户端重启）
  └── 失败处理：最多重试3次，之后放弃

定时拉取（兜底）
  ├── ServiceInfoUpdateService：自适应间隔（6s~60s）
  ├── 触发条件：推送失败 或 定时到期
  └── 保证：即使推送全部失败，最终也能感知变更

两者配合 = 实时性 + 可靠性
```

### 5.3 为什么配置推送只发通知不发内容？

```
错误设计（假设）：推送时直接携带配置内容
  └── 问题：推送是异步的，客户端收到时内容可能已经再次变更
      → 客户端拿到的是"过时"的内容

正确设计（实际）：推送只发 ConfigChangeNotifyRequest（不含内容）
  └── 客户端收到通知后，主动发起 ConfigQueryRequest 拉取最新内容
      → 客户端拿到的一定是"当前最新"的内容

本质：推送是"触发器"，拉取是"获取数据"，两者分离
```

### 5.4 为什么 PushDelayTask 要延迟 500ms？

```
场景：服务提供者批量上线（如滚动发布，10个实例依次启动）

无延迟合并（假设）：
  实例1上线 → 立即推送给100个消费者（100次推送）
  实例2上线 → 立即推送给100个消费者（100次推送）
  ...
  实例10上线 → 立即推送给100个消费者（100次推送）
  总计：1000次推送

有延迟合并（实际）：
  实例1~10在500ms内依次上线
  → 合并为1个 PushDelayTask
  → 只推送1次（100次推送）
  总计：100次推送（减少90%）
```

---

## 6. 2.x vs 1.x 核心差异

| 维度 | 1.x | 2.x | 改进点 |
|------|-----|-----|--------|
| **通信协议** | HTTP 长轮询 | gRPC 双向流 | 实时性：秒级→毫秒级 |
| **连接模型** | 无状态 HTTP | 有状态 gRPC 长连接 | 连接断开即删除临时实例 |
| **配置推送** | 客户端 29.5s 长轮询 | 服务端主动 gRPC Push | 推送延迟大幅降低 |
| **心跳机制** | 客户端定时 HTTP 心跳 | gRPC 连接保活（keepalive） | 减少心跳请求数量 |
| **实例健康** | 心跳超时标记不健康 | 连接断开立即删除 | 故障感知更快 |
| **集群通信** | HTTP 接口 | gRPC（端口9849） | 性能提升 |
| **能力协商** | 无 | `ClientAbilities`/`ServerAbilities` | 版本兼容性更好 |
| **TPS 控制** | 无 | 滑动窗口 + SPI 扩展 | 防止服务端过载 |
| **存储层** | 仅 MySQL | Derby（嵌入式）+ MySQL | 降低部署门槛 |
| **一致性** | Raft（全部数据） | JRaft（CP）+ Distro（AP） | 临时实例性能大幅提升 |

### 1.x → 2.x 迁移关键点

```
客户端升级：
  ├── SDK 版本：nacos-client 1.x → 2.x
  ├── 新增端口：需要开放 9848（gRPC）
  └── 兼容性：2.x 服务端兼容 1.x 客户端（HTTP 接口保留）

服务端升级：
  ├── 新增 gRPC 端口：9848、9849
  ├── 存储：Derby 嵌入式数据库（无需额外安装）
  └── 集群：Distro 协议替代原有 AP 实现
```

---

*文档生成时间：2026-03-05*  
*对应源码版本：Nacos 2.x*  
*上一篇：[00-reading-guide.md](./00-reading-guide.md) | 下一篇：[01-startup-flow.md](./01-startup-flow.md)*
