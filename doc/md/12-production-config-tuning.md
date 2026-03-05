# 第12章：生产环境核心配置与调优

> **源码版本**：Nacos 2.x  
> **核心文件**：`distribution/conf/application.properties`、各模块 `GlobalExecutor.java`、`MetricsMonitor.java`

---

## 核心问题

1. **生产部署 Nacos 集群，最少需要几个节点？为什么？**
2. **MySQL 主从切换是怎么实现的？HikariCP 连接池怎么配？**
3. **Nacos 有哪些关键线程池？如何根据机器规格调整？**
4. **Distro 和 JRaft 各有哪些可调参数？调错了会怎样？**
5. **生产环境必须做哪些安全加固？**
6. **如何通过 Prometheus 监控 Nacos 健康状态？**

---

## 第 1 部分：集群部署规划

### 1.1 节点数量选择

**问题推导**：Nacos 集群使用 JRaft（Raft 协议），Raft 要求超过半数节点存活才能正常工作，节点数量如何选择？

| 节点数 | 可容忍故障节点数 | 是否推荐 | 原因 |
|--------|----------------|---------|------|
| 1 | 0 | ❌ | 单点故障，无高可用 |
| 2 | 0 | ❌ | 任意一个故障即不可用（需要 2/2 存活） |
| **3** | **1** | **✅ 最小推荐** | 1 个故障仍可用（2/3 存活） |
| 4 | 1 | ⚠️ 不推荐 | 容错能力与 3 节点相同，浪费资源 |
| **5** | **2** | **✅ 生产推荐** | 2 个故障仍可用（3/5 存活） |
| 7 | 3 | ✅ 大规模 | 超大规模集群 |

**结论**：生产环境推荐 **3 节点**（最小高可用）或 **5 节点**（更高容错）。**偶数节点没有意义**，不要用 2/4/6 节点。

### 1.2 cluster.conf 配置

```bash
# ${nacos.home}/conf/cluster.conf
# 格式：IP:port（port 为 HTTP 端口，默认 8848）
192.168.1.1:8848
192.168.1.2:8848
192.168.1.3:8848
```

**注意事项**：
- 所有节点的 `cluster.conf` 内容必须完全一致
- 节点 IP 必须是各节点能互相访问的 IP（不能用 `127.0.0.1`）
- 修改 `cluster.conf` 后需要重启节点生效

### 1.3 端口规划

| 端口 | 用途 | 配置项 |
|------|------|--------|
| **8848** | HTTP API + 控制台 | `server.port=8848` |
| **9848** | gRPC 客户端连接（SDK） | `server.port + 1000`，不可单独配置 |
| **9849** | gRPC 集群内部通信 | `server.port + 1001`，不可单独配置 |
| **7848** | JRaft 选举通信 | `server.port - 1000`，不可单独配置 |

**⭐ 重要**：gRPC 端口 = HTTP 端口 + 1000，这是**硬编码**的偏移量，防火墙规则必须同时放开 8848、9848、9849 三个端口。

---

## 第 2 部分：数据库配置

### 2.1 MySQL 生产配置

```properties
# application.properties

# 使用 MySQL（必须显式声明）
spring.datasource.platform=mysql

# 主从双数据库（主库在前）
db.num=2
db.url.0=jdbc:mysql://master-host:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.url.1=jdbc:mysql://slave-host:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=nacos
db.password.0=nacos_password
db.user.1=nacos_readonly
db.password.1=readonly_password
```

**主库选举机制**（源码 `ExternalDataSourceServiceImpl.SelectMasterTask`）：

```java
// 每 10 秒执行一次，遍历所有数据源，尝试执行写操作
// 能写入的即为主库，切换 JdbcTemplate 指向主库
testMasterJT.update("DELETE FROM config_info WHERE data_id='com.alibaba.nacos.testMasterDB'");
```

**注意**：Nacos 通过执行一条 DELETE 语句来探测主库，这条语句会在 `config_info` 表中留下一条 `data_id='com.alibaba.nacos.testMasterDB'` 的记录（如果 DELETE 失败则说明是从库）。这是正常现象，不是 Bug。

### 2.2 HikariCP 连接池调优

```properties
# 连接池配置（源码默认值）
db.pool.config.connectionTimeout=30000    # 获取连接超时，30 秒
db.pool.config.validationTimeout=10000   # 连接验证超时，10 秒
db.pool.config.maximumPoolSize=20        # 最大连接数，默认 20
db.pool.config.minimumIdle=2             # 最小空闲连接数，默认 2
```

**生产调优建议**：

| 参数 | 默认值 | 生产建议 | 说明 |
|------|--------|---------|------|
| `maximumPoolSize` | 20 | 20~50 | 根据 MySQL 最大连接数和 Nacos 节点数计算：`MySQL最大连接数 / Nacos节点数 * 0.8` |
| `minimumIdle` | 2 | 5~10 | 避免冷启动时频繁创建连接 |
| `connectionTimeout` | 30000 | 5000 | 生产环境应更快失败，避免请求堆积 |
| `idleTimeout` | 未配置 | 600000 | 空闲连接 10 分钟后回收 |

### 2.3 数据库初始化

```bash
# 首次部署必须手动初始化 MySQL Schema
mysql -u root -p nacos < ${nacos.home}/conf/mysql-schema.sql

# 主要表结构
# config_info          - 配置主表
# config_info_beta     - 灰度配置
# config_info_tag      - 标签配置
# his_config_info      - 配置历史（变更记录）
# users                - 用户表（鉴权）
# roles                - 角色表（鉴权）
# permissions          - 权限表（鉴权）
```

---

## 第 3 部分：安全加固（生产必做）

### 3.1 三大安全配置

**⭐⭐⭐ 生产环境必须完成以下三项，否则存在严重安全风险：**

```properties
# ① 开启鉴权（默认关闭！）
nacos.core.auth.enabled=true

# ② 自定义 JWT 密钥（默认密钥是公开的！）
# 必须是 Base64 编码的字符串，原始密钥长度 >= 32 字节
# 生成命令：echo -n "your-secret-key-at-least-32-chars" | base64
nacos.core.auth.plugin.nacos.token.secret.key=<自定义Base64密钥>

# ③ 关闭 User-Agent 白名单（历史遗留漏洞，默认已关闭）
nacos.core.auth.enable.userAgentAuthWhite=false
```

**为什么默认密钥危险？**

```properties
# 源码中的默认密钥（所有人都知道！）
nacos.core.auth.plugin.nacos.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789
```

攻击者可以用这个公开密钥伪造任意用户的 JWT Token，直接绕过鉴权。

### 3.2 集群节点身份认证

```properties
# 集群节点间通信的身份标识（防止非法节点加入集群）
nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=<自定义值>
```

**注意**：集群所有节点的 `identity.key` 和 `identity.value` 必须完全一致。

### 3.3 Token 有效期配置

```properties
# JWT Token 有效期，单位：秒，默认 18000 秒（5 小时）
nacos.core.auth.plugin.nacos.token.expire.seconds=18000

# 鉴权信息缓存（开启后，权限变更有 15 秒延迟，但减少 DB 查询）
nacos.core.auth.caching.enabled=true
```

### 3.4 网络访问控制

```properties
# 不对外暴露 actuator 端点（默认已限制）
# 如需 Prometheus 监控，只暴露 prometheus 端点
management.endpoints.web.exposure.include=prometheus,health

# 访问日志（生产建议开启，用于审计）
server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D %{User-Agent}i %{Request-Source}i
```

---

## 第 4 部分：关键线程池调优

### 4.1 线程池全景

Nacos 各模块的关键线程池（源码 `ConfigExecutor.java`、`naming/GlobalExecutor.java`）：

| 线程池名称 | 线程数 | 作用 |
|-----------|--------|------|
| `nacos-grpc-executor` | `getSuitableThreadCount(16)` | gRPC 请求处理（核心） |
| `com.alibaba.nacos.config.server.timer` | 10 | 配置定时任务 |
| `com.alibaba.nacos.config.AsyncNotifyService` | 100 | 配置变更异步通知 |
| `com.alibaba.nacos.config.LongPolling` | 1 | 长轮询调度（单线程） |
| `com.alibaba.nacos.naming.timer` | `cores * 2` | Naming 定时任务 |
| `com.alibaba.nacos.naming.mysql.checker` | `cores * 0.5` | MySQL 健康检查 |
| `com.alibaba.nacos.naming.supersense.checker` | `cores * 0.5` | TCP 健康检查 |
| `com.alibaba.nacos.naming.health` | `cores * 0.5` | 健康检查调度 |

### 4.2 gRPC 线程池计算规则

**源码**（`ThreadUtils.getSuitableThreadCount()`）：

```java
// 算法：找第一个 >= coreCount * multiple 的 2 的幂次
public static int getSuitableThreadCount(int threadMultiple) {
    final int coreCount = PropertyUtils.getProcessorsCount();
    int workerCount = 1;
    while (workerCount < coreCount * threadMultiple) {
        workerCount <<= 1;
    }
    return workerCount;
}
```

**不同机器规格的 gRPC 线程数**：

| CPU 核数 | multiple=16 | 计算过程 | 最终线程数 |
|---------|-------------|---------|-----------|
| 4 核 | 16 | 4×16=64，找 ≥64 的 2 的幂 | **64** |
| 8 核 | 16 | 8×16=128，找 ≥128 的 2 的幂 | **128** |
| 16 核 | 16 | 16×16=256，找 ≥256 的 2 的幂 | **256** |
| 32 核 | 16 | 32×16=512，找 ≥512 的 2 的幂 | **512** |

**调整 gRPC 线程数**（JVM 参数）：

```bash
# 调整 multiple 倍数（默认 16）
-Dremote.executor.times.of.processors=8   # 降低到 8 倍（资源受限时）

# 调整队列大小（默认 16384）
-Dremote.executor.queue.size=8192
```

### 4.3 Naming 健康检查线程数

```java
// naming/GlobalExecutor.java
// 健康检查线程数：取 JVM 参数和 CPU 核数一半的较大值
Integer.max(
    Integer.getInteger("com.alibaba.nacos.naming.health.thread.num", DEFAULT_THREAD_COUNT),
    DEFAULT_THREAD_COUNT  // = cores * 0.5
)
```

**调整方式**：

```bash
# JVM 参数调整健康检查线程数（大量持久实例时需要增加）
-Dcom.alibaba.nacos.naming.health.thread.num=16
```

---

## 第 5 部分：JRaft 参数调优

### 5.1 核心 JRaft 参数

```properties
# JRaft 选举超时（默认 5 秒）
# 网络延迟高时适当增大，避免频繁重选举
nacos.core.protocol.raft.data.election_timeout_ms=5000

# Snapshot 执行间隔（默认 30 分钟）
# 减小可加快新节点同步速度，但增加磁盘 I/O
nacos.core.protocol.raft.data.snapshot_interval_secs=1800

# Raft 内部工作线程数（默认 8）
nacos.core.protocol.raft.data.core_thread_num=8

# Raft 业务请求处理线程数（默认 4）
nacos.core.protocol.raft.data.cli_service_thread_num=4

# 线性读策略（默认 ReadOnlySafe，通过心跳确认 Leader 任期）
# ReadOnlyLeaseBased：基于租约的读，性能更高但可能读到旧数据
nacos.core.protocol.raft.data.read_index_type=ReadOnlySafe

# RPC 请求超时（默认 5 秒）
nacos.core.protocol.raft.data.rpc_request_timeout_ms=5000
```

### 5.2 JRaft 参数调优场景

| 场景 | 问题现象 | 调整建议 |
|------|---------|---------|
| 网络延迟高（跨机房） | 频繁重选举，Leader 不稳定 | 增大 `election_timeout_ms`（10000~15000） |
| 新节点加入慢 | 新节点长时间追日志 | 减小 `snapshot_interval_secs`（300~600） |
| 写入 TPS 高 | Raft 日志积压 | 增大 `core_thread_num`（16~32） |
| 读性能要求高 | 读请求延迟高 | 改为 `ReadOnlyLeaseBased`（牺牲强一致性） |

---

## 第 6 部分：Distro 参数调优

### 6.1 核心 Distro 参数

```properties
# 数据同步延迟（默认 1 秒）
# 同一 key 的多次变更会合并，减少同步次数
nacos.core.protocol.distro.data.sync.delayMs=1000

# 单次同步超时（默认 3 秒）
nacos.core.protocol.distro.data.sync.timeoutMs=3000

# 同步失败重试延迟（默认 3 秒）
nacos.core.protocol.distro.data.sync.retryDelayMs=3000

# 数据校验间隔（默认 5 秒）
# 定期校验各节点数据是否一致
nacos.core.protocol.distro.data.verify.intervalMs=5000

# 单次校验超时（默认 3 秒）
nacos.core.protocol.distro.data.verify.timeoutMs=3000

# 启动时加载数据失败重试延迟（默认 30 秒）
nacos.core.protocol.distro.data.load.retryDelayMs=30000
```

### 6.2 Distro 参数调优场景

| 场景 | 问题现象 | 调整建议 |
|------|---------|---------|
| 服务注册/注销频繁 | 同步流量大，网络压力高 | 增大 `sync.delayMs`（2000~5000），合并更多变更 |
| 集群节点间网络慢 | 同步超时，数据不一致 | 增大 `sync.timeoutMs`（5000~10000） |
| 新节点启动慢 | 数据加载超时 | 减小 `load.retryDelayMs`（10000） |

---

## 第 7 部分：Naming 模块参数

### 7.1 推送参数

```properties
# 服务变更后延迟推送时间（默认 500ms）
# 合并短时间内的多次变更，避免推送风暴
nacos.naming.push.pushTaskDelay=500

# 推送任务超时（默认 5 秒）
nacos.naming.push.pushTaskTimeout=5000

# 推送失败重试延迟（默认 1 秒）
nacos.naming.push.pushTaskRetryDelay=1000
```

### 7.2 客户端过期清理

```properties
# 非活跃客户端过期时间（默认 180 秒 = 3 分钟）
# gRPC 连接断开后，超过此时间才清理客户端数据
nacos.naming.client.expired.time=180000

# 空服务清理间隔（默认 60 秒）
nacos.naming.clean.empty-service.interval=60000

# 空服务过期时间（默认 60 秒）
nacos.naming.clean.empty-service.expired-time=60000

# 元数据清理间隔（默认 5 秒）
nacos.naming.clean.expired-metadata.interval=5000

# 元数据过期时间（默认 60 秒）
nacos.naming.clean.expired-metadata.expired-time=60000
```

### 7.3 推送参数调优场景

| 场景 | 问题现象 | 调整建议 |
|------|---------|---------|
| 服务频繁上下线 | 推送风暴，客户端频繁刷新 | 增大 `pushTaskDelay`（1000~2000ms） |
| 推送延迟要求高 | 服务变更后客户端感知慢 | 减小 `pushTaskDelay`（100~200ms） |
| 网络不稳定 | 推送失败率高 | 增大 `pushTaskTimeout`（10000ms） |

---

## 第 8 部分：JVM 调优

### 8.1 推荐 JVM 参数

```bash
# startup.sh 中的 JVM 参数（根据机器规格调整）

# 4C8G 机器（小规模）
JAVA_OPT="${JAVA_OPT} -Xms2g -Xmx2g -Xmn1g"

# 8C16G 机器（中等规模，推荐）
JAVA_OPT="${JAVA_OPT} -Xms4g -Xmx4g -Xmn2g"

# 16C32G 机器（大规模）
JAVA_OPT="${JAVA_OPT} -Xms8g -Xmx8g -Xmn4g"

# GC 策略（推荐 G1）
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC"
JAVA_OPT="${JAVA_OPT} -XX:MaxGCPauseMillis=200"
JAVA_OPT="${JAVA_OPT} -XX:InitiatingHeapOccupancyPercent=35"

# OOM 时 dump 堆内存（便于排查）
JAVA_OPT="${JAVA_OPT} -XX:+HeapDumpOnOutOfMemoryError"
JAVA_OPT="${JAVA_OPT} -XX:HeapDumpPath=${BASE_DIR}/logs/java_heapdump.hprof"

# GC 日志（便于排查 GC 问题）
JAVA_OPT="${JAVA_OPT} -Xlog:gc*:file=${BASE_DIR}/logs/nacos_gc.log:time,tags:filecount=10,filesize=100m"
```

### 8.2 内存分配原则

| 组件 | 内存占用 | 说明 |
|------|---------|------|
| 配置内存缓存（CacheItem） | 每个配置约 1KB | 10 万配置约 100MB |
| 服务实例内存（ClientManager） | 每个实例约 2KB | 10 万实例约 200MB |
| JRaft 日志缓冲 | 默认 256MB | 可通过 `raft.log.storage.type` 调整 |
| gRPC 线程栈 | 每线程约 512KB | 256 线程约 128MB |

---

## 第 9 部分：监控与告警

### 9.1 开启 Prometheus 监控

```properties
# application.properties
# 暴露 Prometheus 端点
management.endpoints.web.exposure.include=prometheus,health,info

# 开启 Prometheus 服务发现支持（Nacos 作为 Prometheus 的服务发现源）
nacos.prometheus.metrics.enabled=true
```

**访问地址**：`http://{nacos-host}:8848/nacos/actuator/prometheus`

### 9.2 核心监控指标

**Naming 模块指标**（源码 `naming/MetricsMonitor.java`）：

| 指标名 | 标签 | 含义 |
|--------|------|------|
| `nacos_monitor{module=naming,name=serviceCount}` | - | 服务总数 |
| `nacos_monitor{module=naming,name=ipCount}` | - | 实例总数 |
| `nacos_monitor{module=naming,name=subscriberCount}` | - | 订阅者总数 |
| `nacos_monitor{module=naming,name=totalPush}` | - | 推送总次数 |
| `nacos_monitor{module=naming,name=failedPush}` | - | 推送失败次数 |
| `nacos_monitor{module=naming,name=maxPushCost}` | - | 最大推送耗时（ms） |
| `nacos_monitor{module=naming,name=avgPushCost}` | - | 平均推送耗时（ms） |
| `nacos_monitor{module=naming,name=leaderStatus}` | - | 是否为 Raft Leader（1=是） |
| `nacos_naming_subscriber{version=v1}` | - | HTTP 长轮询订阅者数 |
| `nacos_naming_subscriber{version=v2}` | - | gRPC 订阅者数 |

**Config 模块指标**（源码 `config/MetricsMonitor.java`）：

| 指标名 | 标签 | 含义 |
|--------|------|------|
| `nacos_monitor{module=config,name=longPolling}` | - | 当前长轮询连接数 |
| `nacos_monitor{module=config,name=configCount}` | - | 配置总数 |
| `nacos_monitor{module=config,name=notifyTask}` | - | 待通知集群任务数 |
| `nacos_monitor{module=config,name=notifyClientTask}` | - | 待通知客户端任务数 |
| `nacos_monitor{module=config,name=dumpTask}` | - | 待 dump 任务数 |
| `nacos_config_subscriber{version=v1}` | - | HTTP 配置订阅者数 |
| `nacos_config_subscriber{version=v2}` | - | gRPC 配置订阅者数 |

**异常指标**：

| 指标名 | 含义 | 告警建议 |
|--------|------|---------|
| `nacos_exception{module=naming,name=disk}` | 磁盘异常次数 | > 0 立即告警 |
| `nacos_exception{module=naming,name=leaderSendBeatFailed}` | Leader 心跳发送失败 | > 0 立即告警 |

### 9.3 Prometheus 告警规则示例

```yaml
# prometheus-rules.yaml
groups:
  - name: nacos
    rules:
      # 推送失败率告警
      - alert: NacosPushFailedHigh
        expr: rate(nacos_monitor{module="naming",name="failedPush"}[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Nacos 推送失败率过高"
          description: "推送失败率 {{ $value }}/s，超过阈值 0.1/s"

      # 实例数骤降告警（可能是大量实例下线）
      - alert: NacosInstanceCountDrop
        expr: delta(nacos_monitor{module="naming",name="ipCount"}[5m]) < -100
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nacos 实例数骤降"
          description: "5 分钟内实例数减少 {{ $value }} 个"

      # 推送延迟告警
      - alert: NacosPushLatencyHigh
        expr: nacos_monitor{module="naming",name="avgPushCost"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Nacos 推送延迟过高"
          description: "平均推送耗时 {{ $value }}ms，超过 1000ms"

      # 配置通知任务积压告警
      - alert: NacosNotifyTaskBacklog
        expr: nacos_monitor{module="config",name="notifyTask"} > 1000
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Nacos 配置通知任务积压"
          description: "待通知任务数 {{ $value }}，超过 1000"
```

---

## 第 10 部分：日志配置

### 10.1 日志文件位置

Nacos 服务端日志默认路径：`${nacos.home}/logs/`

| 日志文件 | 内容 |
|---------|------|
| `nacos.log` | 主日志（启动、错误） |
| `config-server.log` | 配置中心操作日志 |
| `naming-server.log` | 服务注册发现日志 |
| `naming-raft.log` | JRaft 协议日志 |
| `naming-distro.log` | Distro 协议日志 |
| `remote.log` | gRPC 通信日志 |
| `access_log.*.txt` | HTTP 访问日志 |

### 10.2 日志级别调整

```bash
# 动态调整日志级别（无需重启）
curl -X POST "http://localhost:8848/nacos/v1/console/server/log?logName=naming-server&logLevel=DEBUG"

# 常用日志名
# naming-server    - 服务注册发现
# config-server    - 配置中心
# naming-raft      - JRaft 协议
# naming-distro    - Distro 协议
# remote           - gRPC 通信
```

---

## 第 11 部分：生产配置完整模板

```properties
#============================================================
# Nacos 生产环境配置模板（基于 Nacos 2.x）
#============================================================

#---------- Spring Boot ----------
server.servlet.contextPath=/nacos
server.port=8848
server.error.include-message=ALWAYS

#---------- 数据库（MySQL）----------
spring.datasource.platform=mysql
db.num=2
db.url.0=jdbc:mysql://master:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.url.1=jdbc:mysql://slave:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=nacos
db.password.0=${DB_PASSWORD}
db.user.1=nacos_readonly
db.password.1=${DB_READONLY_PASSWORD}

# HikariCP 连接池
db.pool.config.connectionTimeout=5000
db.pool.config.validationTimeout=3000
db.pool.config.maximumPoolSize=30
db.pool.config.minimumIdle=5

#---------- 安全鉴权（必须配置！）----------
nacos.core.auth.enabled=true
nacos.core.auth.system.type=nacos
nacos.core.auth.plugin.nacos.token.secret.key=${AUTH_SECRET_KEY}
nacos.core.auth.plugin.nacos.token.expire.seconds=18000
nacos.core.auth.caching.enabled=true
nacos.core.auth.enable.userAgentAuthWhite=false
nacos.core.auth.server.identity.key=nacosClusterIdentity
nacos.core.auth.server.identity.value=${CLUSTER_IDENTITY_VALUE}

#---------- Naming 模块 ----------
nacos.naming.data.warmup=true
nacos.naming.push.pushTaskDelay=500
nacos.naming.push.pushTaskTimeout=5000
nacos.naming.push.pushTaskRetryDelay=1000
nacos.naming.client.expired.time=180000

#---------- JRaft ----------
nacos.core.protocol.raft.data.election_timeout_ms=5000
nacos.core.protocol.raft.data.snapshot_interval_secs=1800
nacos.core.protocol.raft.data.core_thread_num=8
nacos.core.protocol.raft.data.rpc_request_timeout_ms=5000

#---------- Distro ----------
nacos.core.protocol.distro.data.sync.delayMs=1000
nacos.core.protocol.distro.data.sync.timeoutMs=3000
nacos.core.protocol.distro.data.verify.intervalMs=5000

#---------- 监控 ----------
management.endpoints.web.exposure.include=prometheus,health,info
nacos.prometheus.metrics.enabled=true

#---------- 访问日志 ----------
server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D %{User-Agent}i %{Request-Source}i
server.tomcat.basedir=file:.
```

---

## 第 12 部分：常见生产问题排查

### 12.1 问题速查表

| 问题现象 | 可能原因 | 排查方向 |
|---------|---------|---------|
| 客户端注册失败 | 防火墙未放开 9848 端口 | 检查 9848 端口连通性 |
| 配置推送延迟高 | `notifyTask` 积压 | 查看 `nacos_monitor{name=notifyTask}` 指标 |
| 服务实例频繁上下线 | 心跳超时参数不合理 | 检查 `heartBeatTimeout`（默认 15s） |
| 集群 Leader 频繁切换 | 网络延迟高 | 增大 `election_timeout_ms` |
| 新节点加入后数据不同步 | Snapshot 间隔太长 | 减小 `snapshot_interval_secs` |
| OOM | 配置/实例数量过多 | 增大堆内存，检查 `ipCount`/`configCount` |
| 推送失败率高 | gRPC 连接不稳定 | 查看 `remote.log`，检查网络 |

### 12.2 健康检查 API

```bash
# 检查节点健康状态
curl http://localhost:8848/nacos/v1/console/health/liveness
# 返回：UP

# 检查集群状态
curl http://localhost:8848/nacos/v1/console/server/state
# 返回：{"standalone_mode":"cluster","function_mode":"All","version":"2.x.x"}

# 查看集群成员列表
curl http://localhost:8848/nacos/v1/core/cluster/nodes
```

---

## 总结

### 生产部署 Checklist

```
□ 集群节点数为奇数（3 或 5）
□ 防火墙放开 8848、9848、9849 端口
□ 使用 MySQL 替代 Derby（生产必须）
□ 初始化 MySQL Schema
□ 开启鉴权（nacos.core.auth.enabled=true）
□ 自定义 JWT 密钥（非默认密钥）
□ 关闭 User-Agent 白名单
□ 配置 HikariCP 连接池参数
□ 根据机器规格调整 JVM 参数（-Xms/-Xmx）
□ 开启 Prometheus 监控端点
□ 配置告警规则（推送失败率、实例数骤降）
□ 开启访问日志
□ 定期备份 MySQL 数据
```

---

*文档生成时间：2026-03-05*  
*对应源码版本：Nacos 2.x*  
*源码参考：`distribution/conf/application.properties`、`naming/MetricsMonitor.java`、`config/MetricsMonitor.java`、`ConfigExecutor.java`、`naming/GlobalExecutor.java`*
