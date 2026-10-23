# 日志链路、Docker 容器化与优雅停机参考手册

# 23. 全链路排障、上下文萃取与日志分流规范 (Structured Logging & Log Sinks)

### 23.1 全链路 TraceID 透传
- 每个外部请求进入网关时，必须分配或提取全局唯一的 `request_id`（推荐采用 **ULID** 或 UUID）；
- 跨 HTTP、gRPC RPC、WebSocket、异步任务全链路显式透传，所有业务日志必须自动携带 `request_id` 字段。

### 23.2 5 大物理日志文件分流标准
生产环境严禁将所有日志混合输出到单个日志文件，必须进行物理分流：

| 日志文件名 | 日志级别 | 核心涵盖场景 | 检索与排障场景 |
| :--- | :---: | :--- | :--- |
| **`access.log`** | `INFO` | HTTP 请求与响应记录（URL、Method、耗时、状态码、IP） | 网关流量分析与接口响应时延审计 |
| **`info.log`** | `INFO` | 核心业务状态流转、状态机跃迁、重要事件通知 | 业务主流程追踪与排障定位 |
| **`error.log`** | `ERROR`| 业务异常、系统捕获的非预期错误、Panic 堆栈 | 线上告警触发与 Bug 溯源 |
| **`slow.log`** | `WARN` | 超过慢查询阈值（如 SQL 超过 200ms、RPC 超过 500ms）的请求 | 性能瓶颈分析与慢 SQL 调优 |
| **`stat.log`** | `INFO` | 容器/服务周期性指标统计（QPS、在线连接数、内存水位） | 容量规划与趋势监控 |

### 23.3 敏感字段动态脱敏 (Data Masking)
- **脱敏红线**：日志打印拦截器必须对敏感数据进行正则掩码脱敏；
- 严禁明文打印：密码（Password）、鉴权令牌（Token/JWT）、API 密钥（SecretKey）、身份证号、银行卡号、手机号（中间 4 位必须打码）。

---


---

# 24. Docker 容器化多环境部署与优雅停机 (Docker & Graceful Shutdown)

### 24.1 多阶段通用 Dockerfile 范式
```dockerfile

---

# 7. 微服务高可用进阶：K8s 优雅停机与上帝文件拆分红线 (Advanced Production Guardrails)

### 7.1 K8s 容器化优雅停机标准 (Graceful Shutdown)
在 Pod 缩容、滚动更新或发布上线时，为避免长连接被强制掐断或处理中的事务中断，必须实现平滑退水：
```go
// 注册进程退出信号拦截
proc.AddShutdownListener(func() {
    logx.Info("Shutdown signal received, waiting for pending tasks...")
    // 1. 停止接受新入请求 / 摘流注册中心 (Nacos/Consul/Etcd)
    // 2. 等待处理中的 HTTP/RPC 请求处理完毕 (超时上限 15s)
    // 3. 关闭异步 Worker 管道消费
    // 4. 释放 Redis 连接与 DB 连接池
    logx.Info("Graceful shutdown completed.")
})
```

### 7.2 上帝文件判定与拆分梯度红线 (Anti-God-Files - 核心红线)
- **三级规模梯度**：
  - **尽量 500 行内**：单一手写 `logic.go` 或 `model.go` 保持健康舒适区，坚守高内聚单一职责；
  - **超过 1000 行酌情拆分**：审视是否存在多职责交叉，优先评估将子流程、复杂状态机流转或私有辅助函数剥离为独立 Helper/子领域服务；
  - **超过 1500 行必须拆分**：触发上帝文件绝对熔断红线，坚决禁止继续无序膨胀，必须拆解；
- **两类免责豁免**：机器自动生成的契约/Model（`types.go`、`*_gen.go`、`*.pb.go`）及单元测试文件（`*_test.go`）不在此限；
- **拆分原则**：
  若某个 Logic 涉及多个子操作（如订单支付完成后的发短信、加积分、履约推送），必须将子操作拆解至专有的独立 Sub-Logic 或领域服务中，禁止在一个方法中堆砌成百上千行代码。

