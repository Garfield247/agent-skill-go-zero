---
name: go-zero-development
description: >-
  Go 语言与 go-zero 高性能微服务架构通用开发与工程规范技能。
  涵盖 goctl 契约生成边界与 Makefile 强制门禁、API / gRPC RPC 设计、
  Handler / Logic / ServiceContext / Model 四层分层职责、
  GORM / MySQL 数据库与按月分表自愈、Redis 缓存与 Lua 分布式锁、
  Goroutine Panic 拦截并发安全、异步批处理 Worker 与长连接网关、
  全链路日志追踪、K8s 优雅停机及上帝文件拆分红线。
---

# Go 语言与 go-zero 微服务开发规范 (Go-Zero Development Skill)

本技能为 AI 代理及研发工程师在基于 **go-zero** 框架开发、重构、审查或扩展微服务时，提供核心架构准则、契约生成门禁与渐进式手册索引。

---

## 1. 核心设计原则 (Core Principles)

1. **既有代码是第一事实来源 (Primary Source of Truth)**：优先对齐当前项目已建立的代码组织架构与命名约定。
2. **坚持 go-zero 官方惯用分层**：严格遵循 `Handler -> Logic -> ServiceContext -> Model` 单向依赖调用链，严禁越级倒灌。
3. 🚨 **严守代码生成边界与 Makefile 强制门禁 (Makefile Gate)**：
   - **坚决禁止在终端直接执行任何裸 `goctl` 命令**（如 `goctl api go`、`goctl rpc proto`、`goctl model`），杜绝手敲参数不一致导致的破坏性覆盖；
   - **强制 Makefile 契约生成 SOP**：
     1. **前置探测**：先检查仓库根目录是否存在 `Makefile` 并通过 `grep -E '^(api|rpc|gen)' Makefile` 锁定标准生成目标；
     2. **统一调用**：强制通过 `run_command("make <target>")`（如 `make gen-api` 或 `make api`）执行生成；
     3. **无 Makefile 兜底**：当且仅当仓库确实无 Makefile 时，才允许向用户出具完整参数说明后降级调用 `goctl`；
   - **自动生成的代码一律视为只读（Read-Only）**，严禁手动修改 Handler / Types / Routes 文件。
4. 🚨 **上帝文件拆分红线 (Anti-God-Files)**：
   - **单文件规模约束**：单一 `logic.go` 或 `model.go` 文件**严禁超过 500 行**，超过必须拆分；
   - **单一职责**：一个 API 接口严格对应一个独立 Logic 文件；复杂编排必须拆分子领域服务或 Helper。
5. **最小化非必要变更 (Minimal Changes)**：恪守原子化修改原则，坚决不做无关重构与大面积全库格式化。
6. **强化防御性编程与防吞异常 (Zero-Swallow)**：严禁使用 `_` 静默丢弃有价值的 error，落实结构化并发、Panic 拦截与安全分布式锁。
7. 🚨 **包名遮蔽防范与公共契约归拢 (`pkg/schema`)**：
   - goctl 生成的 DTO 专属收敛于 `internal/types`（只读）；
   - 跨服务/跨模块通用的通用模型（如分页 `PageReq`/`PageResult[T]`、通用值对象、通用过滤）**坚决禁止命名为 `pkg/types`**（防止在 Logic 中被迫引入 `pkgtypes` 别名地狱与认知混乱），必须统一收敛至 **`pkg/schema/`**（或事件通信流收敛至 `pkg/protocol/`）。

---

## 2. 四层分层职责划分 (Layer Responsibilities)

```text
Client (HTTP/JSON) 
  ↓
API Handler (internal/handler/ - 参数反序列化、基础校验、统一响应，由 goctl 只读维护)
  ↓
API Logic (internal/logic/ - 业务规则编排、事务、权限、日志，手写核心业务区)
  ↓
ServiceContext (internal/svc/ - 持有 DB / Redis / RPC Client 单例连接句柄)
  ↓
Model / Remote RPC (数据存储 / 下游服务)
```

- **Handler 控制层**：只做参数解析、基础验证并交由 Logic 处理，严禁直接包含业务逻辑，严禁直接访问 DB/Redis；
- **Logic 业务层**：单一入口方法，强制使用强类型枚举，统一通过 `l.svcCtx` 获取外部依赖，严禁定义包级全局可变变量；
- **ServiceContext 依赖注入**：全局资源只在 `NewServiceContext` 中初始化一次，严禁在业务请求期间并发新建连接池；
- **Model 持久化层**：负责数据存取与防 SQL 注入，严禁暴露裸连接或无条件全表更新。

---

## 3. 标准开发工作流 (Standard Workflow)

1. **改契约**：在 `.api` 或 `.proto` 文件中增改路由、入参出参结构体；
2. **生代码**：运行 `make <target>`（如 `make gen-api`）更新 Handler、Types 与路由绑定；
3. **审差异**：立即执行 `git diff` 审查自动生成文件的变动，确认无破坏性覆盖；
4. **写业务**：在对应的 `internal/logic/` 文件中填充核心业务逻辑并补齐单元测试；
5. **跑检验**：执行 `go test -v ./internal/logic/...` 确认测试通过与零 Panic。

---

## 4. 渐进式专业参考手册 (Progressive Disclosure References)

根据具体涉及的业务场景，**按需查阅 `references/` 目录下的专门规约**，严禁将全部手册一次性无脑全量加载：

| 领域模块 | 参考子文档 | 核心涵盖规范 |
| :--- | :--- | :--- |
| **API 与契约设计** | [01_api_and_routing.md](references/01_api_and_routing.md) | • 路由版本与动词规范<br>• 统一标准响应体与业务错误码体系<br>• 64 位整型防精度丢失 (`json:",string"`)<br>• RPC / zrpc 跨服务通信向前兼容 |
| **数据库与持久化** | [02_database_and_gorm.md](references/02_database_and_gorm.md) | • GORM 防注入、零值陷阱与事务闭包<br>• 数据库按月分表规范与冷启动自愈机制<br>• 软删除与唯一键冲突原子复活协议<br>• goctl model 临时目录剥离与缓存一致性 |
| **领域枚举与零值安全** | [08_domain_enums_system.md](references/08_domain_enums_system.md) | • 存储数值/代码强类型/外层语义化三层映射<br>• 二元 0/1 与多元从 1 开始零值安全两分法则<br>• 领域枚举标准六件套与 JSON/SQL 驱动实现<br>• 高内聚业务谓词封装与全系统字典出口 |
| **高并发缓存与分布式锁** | [03_redis_and_locking.md](references/03_redis_and_locking.md) | • 环境感知 Key 命名空间 (`env:service:biz:id`)<br>• 强 TTL 约束与缓存三灾防范<br>• UUID + Lua 脚本原子安全分布式锁标准 |
| **异步 Worker 与长连接网关** | [04_workers_and_gateway.md](references/04_workers_and_gateway.md) | • 批量异步削峰双触发刷盘流水线<br>• Redis ZSet 零 MySQL 慢查定时巡检引擎<br>• 单调递增 SeqID 发号器与冷启动自愈<br>• 客户端 3 步 ACK 闭环与互斥踢下线 |
| **并发安全与错误处理** | [05_concurrency_and_errors.md](references/05_concurrency_and_errors.md) | • 防吞异常 5 大场景与标准化处置<br>• Context 上下文传递与强类型萃取 (`pkg/ctxdata`)<br>• Goroutine Panic 安全拦截与循环变量闭包捕获 |
| **日志排障与 DevOps 交付** | [06_logging_and_devops.md](references/06_logging_and_devops.md) | • 全链路 TraceID 透传与 5 大物理日志分流<br>• 多阶段瘦身 Dockerfile 范式与 PID 1 信号转发<br>• K8s 容器化优雅停机与平滑退水标准 |
| **调优武器库与交付基准** | [07_troubleshooting_and_dod.md](references/07_troubleshooting_and_dod.md) | • 数据竞态 (Data Race) 与协程泄漏排查<br>• 慢查询与分布式死锁诊断<br>• AI 常见避坑清单与交付完成定义 (DoD) |
