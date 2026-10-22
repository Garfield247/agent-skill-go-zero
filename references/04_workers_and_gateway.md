# 异步批处理与长连接网关参考手册

# 21. 异步 Worker 服务的控制面与高吞吐批处理引擎

> 🚨 **架构规范**：
> 异步 Worker / 后台巡检 Daemon 进程在生产架构中**绝不允许退化为孤立的裸脚本进程**，必须具备标准的 HTTP 控制面契约（`worker.api`）与生产级批处理引擎。

### 21.1 容器健康探针与控制面
1. **健康检查端点**：
   - `/worker/health`、`/worker/ping`：为 Kubernetes 提供存活（Liveness）与就绪（Readiness）探针支持，异常时自动重启自愈；
2. **运维管理与手动触发端点**：
   - `/worker/admin/trigger/archive`：线上任务积压时，供运维手动强制触发巡检归档；
   - `/worker/admin/trigger/flush`：紧急停机前，手动触发内存通道残余批量刷盘落库；
   - `/worker/admin/trigger/table-provision`：手动预建指定月份的物理分表。

### 21.2 高吞吐批量异步削峰刷盘流水线 (Double-Trigger Batch Flusher)
针对高频写入场景（如实时聊天消息、行为埋点、操作流水），单条直接打库会打满 MySQL 连接池并导致 IOPS 耗尽。必须采用**双触发批量刷盘流水线**：

```text
生产写入 (Logic / RPC)
  ↓ 非阻塞投递
内存缓冲 Channel (chan Item, 容量 10000)
  ↓
Worker Batch Flusher
  ├── 条件 1: 缓冲条数达到 batch_size (如 200 条) → 触发批量 INSERT
  └── 条件 2: 计时达到 flush_interval (如 200ms) → 即使不足 200 条也强制落库
```

- **优雅停机排空 (Graceful Drain & Flush)**：服务收到 `SIGTERM` 时，先关闭输入 Channel，将当前缓冲区内的残余消息全部批量写入数据库后再退出，**实现零数据丢失**。

### 21.3 零 MySQL 慢查的定时巡检引擎 (Zero-Slow-Query with Redis ZSet)
- **痛点反模式**：严禁在数百万级的大表上运行周期性定时扫描（如 `SELECT ... WHERE status = 1 AND updated_at < ...`），这会导致严重的慢查询与行锁冲突；
- **标准范式**：使用 **Redis ZSet 时间轴索引**（Member 为业务主键，Score 为到期截止时间戳）：
  - 定时轮询调用 `ZRangeByScoreWithScores` 拉取已超时的少量任务（如 `LIMIT 0, 100`）；
  - 业务处理完成（如状态变更、资源释放）后，原子调用 `ZRem` 移出时间轴；
  - 全程对 MySQL 保持零周期性慢查。

---


---

# 22. 长连接网关与实时通信架构规范 (Real-Time Gateway & Messaging)

针对 WebSocket 长连接、IM 实时聊天或实时状态机推送服务：

### 22.1 单调递增序号发号器与冷启动自愈 (Monotonic SeqID & Recovery)
- **严格保序需求**：同一个会话或频道的连续消息，必须具备绝对严格单调递增的序列号（`seq_id`），防范网络乱序；
- **Redis INCR 驱动**：正常状态下通过 Redis 原生 `INCR {project}:seq:{channel_id}` 高性能发号；
- **DB 回源原子冷启动自愈 (Recovery)**：
  - 若 Redis 发生内存淘汰、服务重启或跨数月后重新激活旧会话，发号器缓存缺失；
  - **严禁直接从 1 开始发号**（会导致序号倒挂或主键冲突！）；
  - 必须查询数据库中当前频道的最大序号 `SELECT COALESCE(MAX(seq_id), 0) FROM messages WHERE channel_id = ?`；
  - 通过 Redis Lua 脚本原子性初始化发号器水位（仅当 Key 不存在时 SET 并附加 TTL），随后继续递增自增，实现无感自愈。

### 22.2 客户端消息幂等与 3 步 ACK 闭环 (3-Step ACK)
1. **客户端生成 `client_msg_id`（UUID）**：作为上行请求幂等凭据，服务端通过 Redis / 唯一索引校验，防止网络超时重试导致消息重复入库；
2. **3 步 ACK 状态机**：
   - **Step 1: Server ACK**：服务端接收并持久化成功，立即返回该消息服务端分配的唯一 ID 与 SeqID（前端状态置为“已发送”）；
   - **Step 2: Peer Delivered ACK**：对端长连接在线并成功推流投递（前端状态置为“已送达”）；
   - **Step 3: Read ACK**：对端用户打开窗口产生已读行为，上报已读游标，更新未读数（前端状态置为“已读”）。

### 22.3 长连接生命周期与多端互斥踢下线 (Kick-out)
1. **多端互斥策略**：当同一账号在另一台新设备建立长连接时，网关检测到已存在旧连接，必须向旧连接发送 `kick_out` 帧（告知原因），随后优雅主动断开旧 Socket，将在线路由指向新连接；
2. **心跳超时与僵尸连接清理**：双向 Ping-Pong 探活，超过 2~3 个心跳周期未收包的连接判定为僵尸连接，服务端强制主动 Close 并清理 Redis 路由表。

---

