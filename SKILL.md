---
name: go-zero-development
description: >-
  Go 语言与 go-zero 高性能微服务架构通用开发与工程规范技能。
  涵盖 goctl 契约生成边界、API / gRPC RPC 设计、Handler / Logic / ServiceContext / Model 分层职责、
  GORM / MySQL / MongoDB / Redis 数据库与缓存最佳实践、Goroutine Panic 拦截与并发安全红线、
  枚举三层映射、单文件行数与上帝文件拆分红线、主流统一响应与分页规范、异步 Worker 控制面、
  防吞异常五大场景、按月分表自愈、Redis Lua 分布式锁、Docker 容器化与 CI/CD 交付标准。
---

# Go 语言与 go-zero 微服务架构开发技能规范 (Go-Zero Development Skill)

## 概述 (Overview)

本技能为 AI 代理（AI Coding Agents）及研发工程师在基于 **go-zero** 框架开发、重构、审查、排查或扩展 Go 语言微服务时，提供严密且符合工业级生产标准的架构设计准则与工程红线。

### 核心设计原则

1. **既有项目代码是第一事实来源 (Primary Source of Truth)**：优先遵循并对齐当前项目已建立的代码组织架构与命名约定。
2. **坚持 Go 与 go-zero 惯用模式 (Idiomatic Patterns)**：采用地道的 Go 语言语法与 go-zero 官方推荐的分层架构。
3. **严守代码生成边界 (Generated Code Boundaries)**：尊重 `goctl` 的代码生成体系，自动生成的代码一律只读，坚决杜绝手工随意篡改。
4. **拒绝上帝文件 (No God Files)**：严格执行单一职责原则，单文件建议 200~300 行，超过 500 行必须重构拆分；一个 API 接口严格对应一个独立的 Logic 文件。
5. **最小化非必要变更 (Minimal Changes)**：恪守原子化修改原则，坚决不做无关重构与大面积全库格式化。
6. **保持向后兼容 (Backward Compatibility)**：除非用户明确要求破坏性变更，否则必须确保对外 API、RPC 与数据结构的向后兼容。
7. **优先复用既有抽象 (Reuse Existing Utilities)**：引入新库、工具函数或数据结构前，必须先在代码库中搜索复用既有实现。
8. **强化防御性编程与防吞异常 (Defensive & Zero-Swallow)**：严禁使用 `_` 静默丢弃有价值的 error，落实结构化并发、Panic 拦截与安全分布式锁。

### 适用技术栈

- **Language**: Go (1.20+)
- **Core Framework**: go-zero, goctl
- **Protocols**: RESTful HTTP API, gRPC / zrpc, Protocol Buffers (v3), WebSocket
- **Data Persistence**: GORM / GORM Gen, goctl model (sqlc), MySQL (utf8mb4), MongoDB
- **Caching & Locking**: Redis (TTL / Lua 分布式锁)
- **Infra & DevOps**: Docker (Multi-stage build), GitLab CI / GitHub Actions, Kubernetes

---

# 1. 通用规则 (General Rules)

## 1.1 修改前先调研 (Inspect Before Modifying)

在动手修改或创建任何代码之前，必须先按以下顺序执行调研：

1. **审视仓库结构**：梳理各微服务的目录划分方式（单体多服务 Monorepo 还是独立服务拆分）。
2. **查阅项目规范**：阅读仓库中的 `README.md`、`AGENTS.md`、`CONTRIBUTING.md` 或项目文档。
3. **定位契约定义**：找到目标接口所在的 `.api`（HTTP 路由）或 `.proto`（RPC 协议）定义文件。
4. **梳理代码分层**：定位对应的 Handler、Logic、ServiceContext 与 Model 层文件。
5. **参考类似实现**：在代码库中检索相似接口或相近业务逻辑的实现范式。
6. **理解依赖与配置**：查阅 `internal/config` 与 `internal/svc`，确认数据库、缓存及下游 RPC 客户端的注入方式。
7. **确认文件可修改性**：辨别目标文件是否由 `goctl` 自动生成，确定修改源头。

严禁在未理解项目全貌前盲目新建文件或臆造抽象层。

```text
【推荐开发路径】
理解既有模式 → 定位同类实现 → 沿用相同规范 → 实施最小化精准修改

【禁止危险路径】
臆造全新架构 → 强行套用抽象 → 大面积重写既有代码
```

## 1.2 现有代码优先级高于通用规则 (Existing Code Has Priority)

当通用的理论最佳实践与当前项目既有的架构习惯产生冲突时：

- **优先遵循当前项目的约定**，保持整个代码库风格的高度统一。
- **严禁擅自进行无关代码现代化**（如无端将既有项目全库语法重构）。
- **发现架构矛盾或坏味道时**：应当在审查总结或说明中主动指出，供人类工程师决策，绝对禁止静默盲改。

> **示例**：如果当前项目在 Logic 层统一使用：
> ```go
> type XxxLogic struct {
>     ctx    context.Context
>     svcCtx *svc.ServiceContext
> }
> ```
> 即使在某些架构理论中更推崇纯接口入参或独立领域服务，也必须严格沿用当前项目的模式编写。

---

# 2. 项目发现与调用链路推演 (Project Discovery)

在开发新功能前，必须在脑海中建立完整的调用链视图：

### HTTP API 调用链路
```text
Client (HTTP/JSON)
  ↓
API Handler (internal/handler/ - 参数反序列化、基础校验、统一响应)
  ↓
API Logic (internal/logic/ - 业务规则编排、事务、权限、日志)
  ↓
ServiceContext (internal/svc/ - 持有 DB / Redis / RPC Client)
  ↓
Model / Remote RPC (数据存储 / 下游服务)
```

### 跨服务 RPC 调用链路
```text
API Logic
  ↓
RPC Client (zrpc client - 接口存根)
  ↓ [gRPC / Protobuf]
RPC Server (internal/server/ - gRPC 服务分发)
  ↓
RPC Logic (internal/logic/ - 服务端核心业务逻辑)
  ↓
Model / Database (持久化层)
```

### 典型工程目录布局
```text
.
├── app/                           # 各独立微服务或主业务模块
│   ├── api-service/               # HTTP API 服务
│   │   ├── desc/                  # API 契约定义目录 (按端/业务领域分拆)
│   │   │   ├── user/              # 面向终端用户的契约 (如 user.api)
│   │   │   ├── admin/             # 面向管理后台的契约 (如 admin.api)
│   │   │   ├── common/            # 公共通用接口契约 (如 common.api)
│   │   │   └── api.api            # 根契约文件 (import 汇聚各子契约)
│   │   ├── etc/                   # 运行配置文件 (yaml)
│   │   ├── internal/
│   │   │   ├── config/            # 配置结构体定义
│   │   │   ├── handler/           # HTTP Handler 路由控制器 (goctl 生成，薄控制器)
│   │   │   ├── logic/             # 核心业务逻辑编排 (一个接口对应一个独立 Logic 文件)
│   │   │   ├── svc/               # 服务依赖注入上下文 (ServiceContext)
│   │   │   └── types/             # 请求/响应 DTO 结构体 (goctl 生成)
│   │   └── service.go             # 服务主入口
│   │
│   ├── rpc-service/               # gRPC 微服务
│   │   ├── desc/                  # Protobuf 契约 (*.proto)
│   │   ├── etc/                   # RPC 运行配置
│   │   ├── internal/              # RPC 内部实现 (logic, server, svc)
│   │   └── client/                # 供外部引用的 RPC Client 存根代码
│   │
│   └── worker-service/            # 异步任务/后台巡检服务 (具备 HTTP 控制面契约)
│
├── pkg/                           # 全局通用基础设施与支撑包
│   ├── enums/                     # 领域枚举强类型包 (按领域拆分独立文件)
│   ├── types/                     # 跨服务共享的内存模型与业务实体
│   ├── errorx/                    # 统一业务异常与错误码体系
│   └── shard/                     # 分表分库路由工具
│
├── model/                         # 数据访问层 (GORM Gen 或 goctl model)
├── scripts/                       # 自动化构建、代码生成、数据库迁移脚本
├── Dockerfile                     # 容器镜像构建文件
├── Makefile                       # 工程构建、代码生成与测试统一入口
└── go.mod                         # Go 模块依赖定义
```

---

# 3. 全局代码组织与单一职责红线 (No God Files)

> 🚨 **核心架构红线**：
> **严禁在工程中塞入任何庞大的单一文件（上帝文件 God File）！**
> 系统全部包与层级（API 契约、`pkg/enums/`、`internal/logic/`、`model/`、中间件等）必须 100% 贯彻职责隔离。

### 3.1 各层级文件拆分标准

1. **`internal/logic/` 业务用例层**：
   - 严格遵循 **一个 API 接口 / 用例 = 一个独立 `*_logic.go` 文件**；
   - 严禁把多个接口逻辑合并写入同一个文件中；
   - 单个 Logic 专注于自身用例的参数校验、数据编排、状态机流转与错误包装。
2. **`desc/` API 契约定义层**：
   - 必须按访问端或业务领域分为独立子目录（如 `desc/user/`, `desc/admin/`, `desc/common/`），杜绝数十个接口平铺堆砌在一个巨型 `.api` 文件中；
   - 根契约 `api.api` 统一通过相对路径 `import` 各子契约文件，由 `make gen-api` 统一驱动编译。
3. **`pkg/enums/` 领域枚举层**：
   - 严禁将全系统所有枚举堆砌进单个 `enums.go`；
   - 必须按业务领域拆分独立文件（如 `order_status.go`, `user_status.go`）；
   - 每个枚举文件必须包含强类型定义、常量组、序列化/反序列化方法及领域专属判定逻辑。
4. **`pkg/types/` 全局结构体与内存模型层**：
   - 必须按数据领域拆分独立文件，禁止所有 Payload 混杂；
   - 结构化数据强制使用强类型结构体承载，严禁使用散装 `map[string]any` 在跨层业务中传递。
5. **`model/` 数据持久化层**：
   - 严格按表名分为生成代码（`*_gen.go` 或 `*_model_gen.go`）与手写业务扩展代码（`*_model.go`）；
   - 严禁跨表混合编写方法，所有自定义业务查询必须写在对应表的非生成扩展文件中。

### 3.2 单文件规模与复杂度软约束
- **建议行数**：常规业务源文件建议控制在 **200 ~ 300 行** 之间；
- **重构阈值**：当单个文件行数超过 **500 行** 时，必须主动审视是否违反单一职责原则，并按照领域子实体或功能职责进行拆分重构。

### 3.3 目录与 Go 包命名分词规范 (Kebab-Case vs. Snake-Case)
- **基础设施与服务层**：微服务根目录、Docker 镜像名、部署产物统一采用中划线 **`kebab-case`**（如 `api-server`, `order-rpc`）；
- **Go 源码层**：package 名、Go 文件名、子目录必须严格使用小写下划线 **`snake_case`** 或小写单单词（如 `order_status.go`, 包名 `wshub`），杜绝大小写混合导致跨操作系统构建问题。

---

# 4. 领域枚举与强类型转换规范 (`pkg/enums/`)

### 4.1 核心三层映射架构
在微服务数据流转中，枚举必须建立严密的三层映射机制：

```text
数据库底层 (MySQL)   Go 业务代码层 (Logic/Model)   外部网络传输 (HTTP/WS JSON)
   tinyint unsigned  ←───────────────→  Go 强类型枚举  ←─────────────────→  英文语义化字符串 (string)
  (1, 2, 3...)                       (StatusActive)                         ("active", "disabled")
```

1. **数据库底层**：强制使用 **`tinyint unsigned`**（`0` 预留为 Unknown，有效业务值从 `1` 开始，禁止魔法数字）；
2. **Go 代码层**：统一收敛在 `pkg/enums/` 下定义强类型（如 `type UserStatus uint8`），实现 `json.Marshaler` / `json.Unmarshaler` 以及 `sql/driver` 的 `Value` / `Scan` 接口，实现自动无感双向转换；
3. **外部传输 (HTTP / WS JSON)**：强制对外呈现为 **英文语义化字符串（`string`）**，易读且防止枚举数值调整破坏前端解析；
4. **严禁魔法数字与魔法字符串**：业务代码中严禁出现 `if status == "active"` 或 `if status == 1`，必须使用强类型枚举方法（如 `userStatus.IsActive()`、`orderStatus.CanCancel()`）。

---

# 5. go-zero API 接口与路由设计规范

## 5.1 API 定义为单一事实来源 (Source of Truth)

- 所有 HTTP 接口的输入、输出、路径及路由分组由 `.api` 文件唯一定义；
- 遵循先改契约文件，再通过 `goctl` / `Makefile` 生成，最后编写 Logic 业务的开发顺序；
- **严禁手动编辑由 goctl 生成的 Handler / Types / Routes 文件**。

## 5.2 路由分层与路径规范
- **外部公网路由**：面向外部终端、前端应用，统一携带版本前缀，如：
  ```
  /web-api/v1/{module}/{action}
  ```
- **内部微服务集群路由**：面向微服务间专用调用管道，强制以内部前缀隔离，如：
  ```
  /inner-api/v1/{service}/{module}/{action}
  ```
  - 严禁向外部公网网关暴露 `/inner-api/v1/*` 路由；
  - 内部接口必须携带 `X-Request-ID` 等上下文跟踪头。
- **路径命名风格**：路由路径中的短语统一使用小写中划线连接 **`/xxxx-xxxx/`**（如 `/user-group/detail`），严禁使用下划线、驼峰或中文拼音。

## 5.3 基础动作动词规范 (Action Verbs)
| 动作动词 | 语义与适用场景 | 外部路由示例 | 内部路由示例 |
| :--- | :--- | :--- | :--- |
| **`index`** | **分页列表查询** | `/web-api/v1/user-group/index` | - |
| **`list`** | **全量列表/字典查询 (不分页)** | `/web-api/v1/common/enums` | `/inner-api/v1/user/list` |
| **`detail` / `info`** | **单个详情查询** | `/web-api/v1/user-group/detail` | `/inner-api/v1/user/info` |
| **`create` / `start`** | **新建资源 / 开启任务** | `/web-api/v1/user-group/create` | `/inner-api/v1/task/start` |
| **`update`** | **更新已有资源** | `/web-api/v1/user-group/update` | - |
| **`delete` / `close`** | **删除 / 结束关闭** | `/web-api/v1/user-group/delete` | `/inner-api/v1/task/close` |
| **`status`** | **状态更新/启停用切换** | `/web-api/v1/user/status` | - |

## 5.4 禁止在 URL Path 中传递业务动态 ID (核心红线)
- **查询类接口（GET）**：业务标识统一使用 Query 参数传递：
  - ✅ 正确：`/web-api/v1/user/detail?user_id=1001`
  - ❌ 严禁：`/web-api/v1/user/detail/1001` 或 `/users/{id}`
- **变更类接口（POST / PUT / DELETE）**：业务 ID 必须包裹在请求 JSON Body 中：
  - ✅ 正确：`POST /web-api/v1/user/update`，Body 为 `{"user_id": "1001", "name": "Alice"}`
  - ❌ 严禁：`PUT /web-api/v1/user/update/1001`

## 5.5 64 位长整型传输与 JS 精度防丢 (方案: `json:",string"`)
- **精度截断风险**：JavaScript 的 `Number.MAX_SAFE_INTEGER` 为 $2^{53} - 1$（16 位），而雪花 ID 与 64 位大整型为 19~20 位。若直接输出数字给前端，前端解析将发生低位截断（末位变 0）。
- **规范标准**：
  - 底层数据库与 Go 结构体统一采用强类型的 `int64 / uint64`；
  - 在 `.api` 定义与对外 DTO 结构体中，长整型 ID 字段 Tag 显式追加 **`,string`**（例如 `json:"id,string"`、`json:"user_id,string"`）；
  - WebSocket 广播与长连接推送帧中，涉及 64 位整型 ID 必须格式化为字符串输出；
  - **短数值字段约束**：普通业务状态码、分页参数（`page`, `page_size`, `status`）严禁盲目加 `,string`。

---

# 6. 主流统一标准响应与分页规范

为保持对外接口规范的高度通用性与传播友好性，全系统采用**主流标准化 REST/JSON 统一响应体系**：

## 6.1 基础标准响应体 (Standard Response Envelope)
所有 HTTP API 响应体均遵循通用标准外层包装：

```json
{
  "code": 0,
  "msg": "ok",
  "data": { ... }
}
```

- **`code`**：业务状态码（整数）。`0`（或部分标准体系中的 `200`）代表业务处理成功，非 `0` 代表特定业务错误；
- **`msg`**：人类可读的提示信息（字符串）；
- **`data`**：业务载荷数据（对象、数组或 null）。

## 6.2 通用标准分页响应体 (Pagination Response)
对于所有分页列表查询接口，`data` 节点内统一包含分页度量元数据与结果集：

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "page": 1,
    "page_size": 20,
    "total": 95,
    "total_pages": 5,
    "list": [
      {
        "id": "358866610846437376",
        "name": "示例记录"
      }
    ]
  }
}
```

### 通用分页字段标准说明
| 字段名称 | 类型 | 业务含义与规范 |
| :--- | :--- | :--- |
| **`page`** | `int` | 当前页码（从 1 开始） |
| **`page_size`** | `int` | 每页条数（对应请求入参 `page_size`） |
| **`total`** | `int64` | 满足条件的总记录条数 |
| **`total_pages`** | `int` | 总页数（$\lceil 	ext{total} / 	ext{page\_size} 
ceil$） |
| **`list`** | `array` | 当前页实体记录列表数组（无数据时返回空数组 `[]`，严禁返回 `null`） |

> **注**：若当前企业既有项目约定使用 Laravel/Django 风格的深度翻页结构（含 `current_page`, `per_page`, `first_page_url`, `last_page_url` 等导航 URL），**严格按既有项目约定为准**。新建立项推荐上述轻量通用标准。

## 6.3 主流统一业务异常与错误码体系 (Mainstream Error Code Standards)

### 6.3.1 核心设计理念
1. **统一 HTTP 200 响应 + 业务状态码**：客户端接口统一返回 `HTTP 200 OK`（或配合网关协议映射），通过 JSON 响应体内的 `code` 明确表达业务结果，**严禁向前端直接裸抛 HTTP 500**；
2. **底层 Error 与对外异常解耦**：代码内部使用 Go 标准 `error` 并通过 `%w` 向上冒泡记录完整调用栈；面向前端时，通过全局异常处理器统一转译为带有 `Code` 和安全提示信息的 `CodeError`；
3. **安全脱敏与防信息泄漏（核心红线）**：严禁向客户端暴露任何底层 SQL 报错（如 `Duplicate entry ...`）、Redis 连接异常、或未捕获的 Panic 堆栈。内部使用 `l.Errorf` 完整记录排障日志，外部一律转译为友好提示。

### 6.3.2 业界主流错误码标准体系

团队可根据工程规模，对齐以下业界最通用的两套错误码标准之一：

#### 方案 A：HTTP 语义映射型标准错误码（RESTful-Aligned，中小微服务首选）
以最广为人知的 HTTP 语义状态码作为业务 `code` 基准，心智负担极低，前后端协作无缝：

| 错误码 (`code`) | 错误标识 (`Status`) | 涵盖业务场景 | 统一提示文案 (`msg`) 示例 |
| :---: | :--- | :--- | :--- |
| **`0`** | **OK** | 请求正常执行完成 | `"ok"` / `"success"` |
| **`400`** | **Bad Request** | 业务前置条件不满足、状态机冲突、非法业务操作 | `"业务处理失败"` / `"当前状态不支持该操作"` |
| **`401`** | **Unauthorized** | 未携带 Token、Token 签名失效、凭证已过期 | `"用户未登录或登录已过期"` |
| **`403`** | **Forbidden** | 账号已被封禁、无权限访问当前功能、租户越权拦截 | `"无访问权限"` |
| **`404`** | **Not Found** | 访问的 API 路由不存在、目标查询的业务实体未找到 | `"请求的资源不存在"` |
| **`409`** | **Conflict** | 资源唯一性冲突（如手机号已注册）、重复提交、幂等冲突 | `"数据已存在或已被占用"` |
| **`422`** | **Unprocessable Entity** | 入参缺失必填字段、正则校验不匹配、范围非法 | `"请求参数不合法"` |
| **`429`** | **Too Many Requests** | 触发用户级或接口级频次限流 | `"请求过于频繁，请稍后再试"` |
| **`500`** | **Internal Server Error**| 数据库底层异常、代码空指针未捕获、系统未知错误 | `"服务开小差了，请稍后重试"` (内部记日志，外发必脱敏) |
| **`502 / 503`** | **Service Unavailable** | 下游 RPC 微服务调用超时、外部第三方服务不可用 | `"网络服务暂时不可用，请稍后重试"` |

#### 方案 B：五位分段式模块化错误码（Alibaba / CNCF 微服务标准，大型服务推荐）
格式规范：`[错误来源 1位][模块编号 2位][具体错误 2位]`：
- **`0`**：成功
- **`10001 ~ 10099`**（客户端通用错误）：如 `10001` 参数校验失败，`10002` 未登录/Token过期，`10003` 无权限访问，`10004` 资源不存在，`10005` 触发限流；
- **`20001 ~ 29999`**（业务逻辑错误）：如 `20101` 用户账号已被冻结，`20201` 订单状态流转非法，`20202` 账户余额或库存不足；
- **`30001 ~ 39999`**（第三方/依赖基础设施错误）：如 `30001` 数据库执行故障，`30002` Redis 缓存故障，`30003` 远程 RPC 微服务超时。

### 6.3.3 go-zero 统一异常拦截落地最佳实践

在工程中统一通过 `pkg/errorx` 封装并在 go-zero 入口注册全局拦截：

```go
// 1. pkg/errorx/error.go - 基础业务异常定义
package errorx

type CodeError struct {
    Code int    `json:"code"`
    Msg  string `json:"msg"`
}

func (e *CodeError) Error() string {
    return e.Msg
}

func NewCodeError(code int, msg string) error {
    return &CodeError{Code: code, Msg: msg}
}

func NewParamError(msg string) error {
    return NewCodeError(422, msg)
}

func NewServerError(msg ...string) error {
    defaultMsg := "服务暂时不可用，请稍后重试"
    if len(msg) > 0 && msg[0] != "" {
        defaultMsg = msg[0]
    }
    return NewCodeError(500, defaultMsg)
}
```

```go
// 2. 在 main.go 服务启动时注册 go-zero 全局统一错误拦截器
httpx.SetErrorHandlerCtx(func(ctx context.Context, err error) (int, any) {
    switch e := err.(type) {
    case *errorx.CodeError:
        // 预期的业务 CodeError: 直接按统一结构返回
        return http.StatusOK, types.BaseResponse{
            Code: e.Code,
            Msg:  e.Msg,
            Data: nil,
        }
    default:
        // 未捕获的原生系统异常/panic: 记录详细上下文堆栈，对外安全脱敏
        logx.WithContext(ctx).Errorf("[GlobalErrorHandler] unhandled internal error: %+v", err)
        return http.StatusOK, types.BaseResponse{
            Code: 500,
            Msg:  "服务开小差了，请稍后重试",
            Data: nil,
        }
    }
})
```

```go
// 3. 在 Logic 业务层中标准调用
if req.Name == "" {
    return nil, errorx.NewParamError("用户名不能为空")
}
if user == nil {
    return nil, errorx.NewCodeError(404, "目标用户不存在")
}
if user.Status != enums.UserStatusActive {
    return nil, errorx.NewCodeError(400, "用户账号已被禁用")
}
```

---

# 7. Handler 控制层规范 (Handler Rules)

Handler 必须保持为“薄控制器”（Thin Handler），其唯一职责是协议适配与流程编排：

```go
func CreateUserHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req types.CreateUserRequest
        // 1. 参数反序列化与基础校验
        if err := httpx.Parse(r, &req); err != nil {
            httpx.ErrorCtx(r.Context(), w, err)
            return
        }

        // 2. 调用 Logic 层
        l := logic.NewCreateUserLogic(r.Context(), svcCtx)
        resp, err := l.CreateUser(&req)
        if err != nil {
            // 3. 统一错误响应处理
            httpx.ErrorCtx(r.Context(), w, err)
            return
        }

        // 4. 成功响应输出
        httpx.OkJsonCtx(r.Context(), w, resp)
    }
}
```

### 核心禁令 (红线)
- **绝对禁止在 Handler 中编写业务逻辑代码**；
- **绝对禁止在 Handler 中直接查询数据库或操作 Redis**；
- **绝对禁止在 Handler 中启动复杂数据库事务**。

---

# 8. Logic 业务层规范 (Logic Layer)

Logic 层是所有核心业务规则编排的聚合中心。

### Logic 方法标准职责
1. 校验业务语义规则（资格前置检查、状态机约束）；
2. 调用持久化 Model 层或下游 RPC 微服务；
3. 执行业务运算与 DTO 转换；
4. 记录带上下文的业务日志（`l.Logger.WithContext(l.ctx)`）；
5. 包装规范的业务错误并返回响应。

### 基础设施隔离红线
- **严禁在 Logic 内部动态初始化基础设施连接**（禁止在函数内调用 `gorm.Open` 或连接 Redis）；
- 所有基础设施客户端必须统一通过 `l.svcCtx` 获取。

---

# 9. ServiceContext 与依赖注入规范 (ServiceContext & DI)

```go
type ServiceContext struct {
    Config    config.Config
    UserModel model.UserModel
    Redis     *redis.Redis
    UserRpc   userclient.User
    DB        *gorm.DB
}
```

- **单例注入**：数据库连接池、Redis 客户端、下游 RPC Client 必须在 `NewServiceContext` 初始化时统一创建，全服务共享；
- **禁止全局可变单例**：严禁在包级别定义 `var DB *gorm.DB`，全局变量会破坏单测 Mock、导致并发竞争与状态污染；
- **通过 Context 消费**：在 Logic 中统一通过 `l.svcCtx.XXX` 引用。

---

# 10. goctl 代码生成铁律 (goctl Guidelines)

1. **Makefile 封装优先**：团队开发中禁止手动裸敲杂乱的 `goctl` 命令行参数，必须通过 `Makefile`（如 `make gen-api` / `make gen-model` / `make gen-rpc`）统一触发；
2. **定制模板对齐**：使用 `./template` 本地定制模板，统一生成代码的文件命名风格（强制小写下划线 `snake_case`）；
3. **安全 5 步流程**：
   - 检查 `goctl --version` 工具版本；
   - 检查 `git status --short` 工作区状态；
   - 确认待变更文件范围；
   - 执行 `make gen-api` / `make gen-rpc`；
   - 立即执行 `git diff` 严格审查，确认无误抹除已有手工逻辑。

---

# 11. 生成代码边界控制 (Generated Code Boundaries)

由 `goctl` 自动生成的代码在日常开发中**一律视为只读（Read-Only）**：

- **典型只读目录/文件**：
  - `internal/handler/`
  - `internal/types/types.go`
  - `internal/server/`
  - `model/*_gen.go`
  - `*client/*.go`
- **修复原则**：生成代码缺失字段或不符预期时，**严禁直接在生成文件中打补丁**，必须修改源契约（`.api` / `.proto` / DDL / 模板）后重新生成。

---

# 12. RPC / zrpc 跨服务通信规范

- **严禁跨服务直连异构数据库**：若某类数据归属外部服务，必须通过该服务的 RPC 客户端经由接口访问，严禁直连对方数据库查询；
- **Proto 契约向前兼容**：
  - 新增字段使用全新 tag 编号，严禁修改已有字段语义或复用已删除编号；
  - 废弃字段必须显式声明 `reserved`：
    ```protobuf
    message UserDetail {
        reserved 5, 8 to 11;
        string name = 1;
        int64 id = 2;
    }
    ```

---

# 13. 数据库与持久化规范 - MySQL

1. **字符集与 Emoji 原生支持**：
   - 数据表字符集必须设为 **`utf8mb4`**，排序规则为 **`utf8mb4_unicode_ci`**，确保各类 Emoji 表情无损存储；
2. **主键与审计字段标准**：
   - 主键统一采用 `id bigint unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY`；
   - 业务表必须包含标准审计字段：`created_at`, `updated_at`, `deleted_at`；
3. **统一持久化选型**：严格使用项目既有抽象（GORM 或 go-zero 内置 sqlc/model），严禁在同一模块混用多套 ORM；
4. **所有表和字段必须具备详尽的中文注释**。

---

# 14. GORM / GORM Gen 工程规范

### 14.1 防 SQL 注入与参数化查询
```go
// 正确：参数化查询
db.WithContext(ctx).Where("user_id = ?", userID).First(&user)

// 严禁：字符串拼接防注入漏洞！
db.Where(fmt.Sprintf("user_id = %d", userID)) // 绝对禁止！
```

### 14.2 零值陷阱与精准更新
- GORM 使用 Struct 更新时默认会**忽略所有零值字段**（`0`, `false`, `""`）：
  ```go
  // 正确：明确指定 Select 字段或使用 map
  db.Model(&user).Select("Age", "Status").Updates(User{Age: 0, Status: false})
  // 或
  db.Model(&user).Updates(map[string]any{"age": 0, "status": false})
  ```

### 14.3 杜绝无条件全表更新或删除
执行 `Delete` 或 `Update` 必须确保带有明确的 `Where` 条件，防范全表被误清空。

### 14.4 事务标准闭包范式
```go
err := db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return fmt.Errorf("create order: %w", err)
    }
    if err := tx.Model(&inventory).Where("item_id = ? AND stock >= ?", itemID, num).
        UpdateColumn("stock", gorm.Expr("stock - ?", num)).Error; err != nil {
        return fmt.Errorf("deduct stock: %w", err)
    }
    return nil
})
```
- **严禁**将单次读操作包装在事务中（无意义消耗 DB 连接池与锁资源）。

---

# 15. 数据库按月分表规范与自愈机制 (Monthly Sharding)

针对日志、流水等单调持续膨胀的大数据量表：
1. **动态表名计算**：通过统一工具函数（如 `GetOpLogTableName(t time.Time)`）按月格式化物理表名（如 `operation_logs_202609`）；
2. **定时预建 + 写入自愈**：
   - 后台 Worker 每月 25 日预建下月物理表；
   - 写入层捕获 MySQL `Error 1146 (Table doesn't exist)` 异常时，自动触发 DDL 补建表并执行重试自愈；
3. **跨月查询约束**：跨月多表查询采用 `UNION ALL` 拼接，业务层必须强制限制跨度（通常不超过 3 个月）。

---

# 16. MongoDB 规范

- 所有 MongoDB 操作首个参数必须显式传递 `ctx`；
- 正确区分“数据不存在”与“系统故障”，将 `mongo.ErrNoDocuments` 转化为明确的业务 NotFound 状态。

---

# 17. Redis 缓存与分布式锁标准

### 17.1 命名空间与多环境隔离规范 (Environment-Aware Key Namespace)

为了兼容**多套环境共用同一 Redis 实例**（如本地联调与测试共用、灰度演练与正式共用）的隔离需求，Redis Key 的命名空间采用分层分段规范：

#### 标准命名格式
```text
{project}:[{env}:]{module}:{object}:{identifier}
```

#### 常见环境标签 (`env`) 枚举与场景
- **`local`**：开发者本地机器独立调试，与局域网共用 Redis 实例时防止污染共享数据；
- **`dev`**：开发联调集成环境；
- **`test`**：测试环境 / QA 自动化流水线测试；
- **`stage` / `long`**：预发布长期演练环境、灰度发布节点（Canary）；
- **`prod`**：正式生产环境。

#### 核心隔离场景与收益
1. **本地测试共用隔离**：多个开发者本地使用同一个内网 Redis 时，增加 `local` 或开发者标识前缀，杜绝相互覆盖彼此的缓存状态；
2. **灰度发布与正式共用隔离 (Gray / Canary Isolation)**：在微服务灰度发布演练或长期预发验证（`long` / `stage`）与正式（`prod`）共用底层 Redis 基础设施时，通过环境标签实现数据无损逻辑隔离，互不踩踏；
3. **独立集群可选省略**：在生产环境拥有物理完全隔离的独立 Redis 集群时，环境标签可选省略（保持 `{project}:{module}:{object}:{identifier}`）；
4. **工程落地最佳实践（配置化注入）**：
   - 严禁在业务 Logic 中到处硬编码环境判断（如 `if env == "dev"`）；
   - 统一在 `config.Config` 中定义 `KeyPrefix: "myproject:dev:"`（通过环境变量或对应配置文件覆盖注入）；
   - 在 `pkg` 或 `ServiceContext` 中提供统一的 Key 格式化构建函数（如 `BuildKey(module, object, id)`），业务层透明无感调用。

### 17.2 TTL（生命周期）约束
- **严禁生成无 TTL 的垃圾缓存**：除少数全局发号序列器外，所有业务缓存必须显式设置过期时间（TTL）；
- 在线长连接或会话路由 Key 兜底设置 TTL（由心跳周期性 `EXPIRE` 续期）。

### 17.3 安全分布式锁标准实现 (UUID + Lua)
- **Lock Key 格式**：`{project}:[{env}:]lock:{business}:{unique_id}`；
  - ⚠️ **环境隔离特别警示**：若测试/灰度与共享环境共用 Redis，分布式锁**必须携带环境前缀**，否则测试环境执行加锁会直接死锁线上业务实体！
- **加锁规则**：必须生成随机 UUID 作为 Value，并附带明确的超时 TTL（防死锁）；
- **解锁规则**：**必须通过 Lua 脚本原子性校验持有者 UUID 后方可删除**，绝对禁止直接使用裸 `DEL` 导致误删他人持有的锁：
  ```lua
  -- 安全释放锁 Lua 脚本
  if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
  else
      return 0
  end
  ```

---

# 18. Go 语言错误处理与防吞异常工程红线 (Zero Blank Identifier for Error)

> 🚨 **核心工程红线**：
> 在全系统所有生产代码中（除 `*_test.go` 外），**一律严禁使用空标识符 `_` 静默忽略 `error` 或具有业务语义的有意义返回值**！

### 18.1 五大场景标准化处置准则

| 场景分类 | 涉及操作 | 处置规范与红线要求 |
| :--- | :--- | :--- |
| **1. 参数解析与数据转换** | `time.Parse`、`strconv.Atoi`、`json.Unmarshal` | **严禁忽略解析错误**。必须显式判断 `err != nil`，记录带上下文日志并返回统一参数异常，严禁非法零值流入业务深层引发全表扫描或数据污染。 |
| **2. 核心主业务 DB/Redis 流转** | 状态流转、扣减容量、已读清零、出入队 | **严禁降级静默忽略**。底层更新失败必须携带上下文记录 `l.Errorf` 并返回系统异常，阻断主流程并触发事务回滚。 |
| **3. 旁路与异步任务落库** | 审计日志写入、缓存预热、事件广播 | **允许主流程继续，但必须显式记日志**。失败必须使用 `l.Errorf("[模块] 描述: err=%v", err)` 留痕，杜绝静默失败无法排障。 |
| **4. 微服务间 Inner-API 调用** | 微服务间 SDK HTTP/RPC 调用 | **严禁抛弃 SDK 返回的 error**。即使是在异步协程中调用，也必须判断并记录带有业务唯一标识的错误日志。 |
| **5. WebSocket 推流与网络 I/O** | 消息向对端推送、在线连接路由更新 | **必须显式处理结果分支**：<br>① 推流返回 `err != nil` 记录 `l.Errorf`；<br>② 推流成功但目标未连接（`count == 0`）记录 `l.Debugf` 标记对端离线；<br>③ 路由更新失败记录 `l.Errorf`。 |

### 18.2 典型代码对比与规范范式

```go
// ❌ 严禁写法：静默忽略数据库更新错误与转换异常
_ = l.svcCtx.OrderModel.UpdateStatus(l.ctx, orderID, status)
startTime, _ = time.ParseInLocation("2006-01-02 15:04:05", req.StartTime, time.Local)
_, _ = l.svcCtx.InnerAPIClient.SyncStatus(l.ctx, &req)

// ✅ 标准写法 1 (数据转换与主流程校验):
startTime, err := time.ParseInLocation("2006-01-02 15:04:05", req.StartTime, time.Local)
if err != nil {
    l.Errorf("[OrderList] parse start_time failed: time=%s, err=%v", req.StartTime, err)
    return nil, errorx.NewParamError("开始时间格式非法")
}

// ✅ 标准写法 2 (核心存储更新安全阻断):
if err := l.svcCtx.OrderModel.UpdateStatus(l.ctx, orderID, status); err != nil {
    l.Errorf("[CloseOrder] update order status failed: order_id=%d, err=%v", orderID, err)
    return nil, errorx.NewServerError("更新订单状态失败")
}

// ✅ 标准写法 3 (旁路调用与网络推流显式留痕):
if _, err := l.svcCtx.InnerAPIClient.SyncStatus(l.ctx, &req); err != nil {
    l.Errorf("[InnerAPI] sync status failed: order_id=%d, err=%v", orderID, err)
}
if count, err := l.svcCtx.WsRouter.PushToTarget(ctx, targetID, payload); err != nil {
    l.Errorf("[Push] push msg failed: target_id=%d, err=%v", targetID, err)
} else if count == 0 {
    l.Debugf("[Push] target is offline: target_id=%d", targetID)
}
```

- 跨层传递错误必须使用 `fmt.Errorf("...: %w", err)` 包装；
- 业务层与 Logic 层中**禁止主动调用 `panic()`**。

---

# 19. Context 上下文传递规范 (Context Propagation)

1. **显式第一参数传递**：所有涉及网络 I/O、RPC 调用、数据库操作的方法，必须将 `ctx context.Context` 作为方法的第一个参数；
2. **严禁上下文断流**：在请求处理链路中，**严禁无故使用 `context.Background()`** 替代上游传递的 `r.Context()` 或 `l.ctx`，否则链路追踪 TraceID 与超时取消彻底失效；
3. **禁止将 Context 存在长期结构体内部**，禁止使用 `context.WithValue` 传递核心业务参数。

---

# 20. 并发控制与 Goroutine 安全红线 (Goroutine Safety)

### 核心红线 1：Goroutine Panic 拦截
所有新启动的后台 Goroutine，**必须在函数首行使用 `defer-recover` 拦截 `panic`**，严防协程崩溃击垮整个微服务进程：

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            logx.WithContext(ctx).Errorf("goroutine panic recovered: %v, stack: %s", r, debug.Stack())
        }
    }()
    // 具体的异步处理逻辑...
}()
```

### 核心红线 2：循环变量闭包捕获
注意循环变量闭包捕获陷阱，并发时必须显式作为参数传入协程：
```go
for _, item := range list {
    go func(val ItemType) {
        process(val)
    }(item)
}
```

- 包含并发逻辑的代码变动，必须执行 `go test -race ./...` 进行竞态检测。

---

# 21. 异步 Worker 服务的控制面标准 (Worker Control Plane)

> 🚨 **架构规范**：
> 异步 Worker / 后台巡检 Daemon 进程在生产架构中**绝不允许退化为孤立的裸脚本进程**，必须具备标准的 HTTP 控制面契约（`worker.api`）。

1. **容器健康检查探针**：
   - `/worker/health`、`/worker/ping`：为 Kubernetes 提供存活（Liveness）与就绪（Readiness）探针支持，确保异常时自动重启自愈；
2. **运维管理与手动触发端点**：
   - `/worker/admin/trigger/archive`：线上任务积压时，供运维手动强制触发巡检归档；
   - `/worker/admin/trigger/flush`：紧急停机前，手动触发内存通道残余消息批量刷盘落库；
   - `/worker/admin/trigger/table-provision`：手动预建指定月份的物理分表；
3. **任务流水线融合**：
   - 依赖项统一通过 `ServiceContext` 管理；
   - 异步计算流水线（Flusher 刷盘、Archiver 归档、Cron 定时）作为常驻 Goroutine 在启动时拉起。

---

# 22. 多租户系统隔离规范 (Multi-Tenant Systems)

1. **租户凭证自闭环**：租户身份（`tenant_id`）必须从认证 Context 中提取，禁止直接信任前端可被篡改的传参；
2. **查询条件必带租户过滤**：业务表增删改查 SQL，必须无条件携带 `tenant_id = ?` 约束，严防跨租户越权数据泄漏。

---

# 23. 性能调优与查询守则 (Performance & SQL Rules)

1. **基于数据与 Profiling 优化**：拒绝凭直觉负优化，善用 `go test -bench`、`pprof`；
2. **数据库性能优先看执行计划**：使用 `EXPLAIN` 检查索引命中（`key`）、扫描行数（`rows`）与临时表使用；
3. **高频业务路径严禁无脑 `SELECT *`**，必须使用 `Select("id", "status", "created_at")` 仅投影必要字段；
4. **防深度分页**：大数据量（数百万级）大表分页，严禁大 Offset，应使用 Keyset/游标分页：
   ```sql
   WHERE id > ? ORDER BY id ASC LIMIT ?
   ```

---

# 24. 单元测试规范 (Testing)

- **优先采用表格驱动测试（Table-Driven Tests）**：覆盖正常流、临界边界、非法输入、空指针防御与错误降级；
- **交付前质量检查命令集**：
  ```bash
  go test ./...        # 运行所有单元测试
  go test -race ./...  # 竞态安全检查 (并发改动必跑)
  gofmt -w .           # 规范化代码格式
  go vet ./...         # 官方静态分析检查
  ```

---

# 25. 代码注释与文档化规范 (Commenting Standards)

1. **包级注释**：每个 package 主文件顶部必须包含中文包级注释，说明职责、导出实体与注意事项；
2. **结构体与字段注释**：说明实体的业务含义；每个字段编写行尾注释，写明含义、单位、取值范围或约束；
3. **函数与方法注释**：第一句话以函数名开头，说明功能意图、关键入参业务语义、返回值及可能返回的错误；
4. **行内关键注释**：在状态机流转、并发加锁、事务边界、算法核心逻辑处解释 **Why & How**，严禁无效废话注释。

---

# 26. 规范代码风格 (Code Style)

1. **卫语句（Guard Clauses）优先**：尽早返回，降低代码圈复杂度与嵌套层级；
2. **拒绝单实现接口滥用**：禁止为了写接口而写单实现接口，仅在存在多实现、跨系统 Mock 或明确架构边界时定义 `interface`。

---

# 27. AI 常见错误与避坑指南 (Common AI Mistakes to Avoid)

AI 代理在编写 go-zero 代码时必须严防以下 8 大典型错误：

1. **擅自篡改底层架构**：把 go-zero 路由强行改成 Gin、Fiber 或 Kratos 风格；
2. **直接手工修改生成代码**：手动修改 `internal/handler/` 或 `*_gen.go`，导致下次生成被全量覆盖；
3. **滥建冗余接口**：为每个 Logic 凭空建立单实现接口；
4. **随处定义全局数据库客户端**：搞出 `var DB *gorm.DB` 全局单例，破坏依赖注入；
5. **对既有通用工具视而不见**：重新造轮子实现既有公共包已有的分页或工具；
6. **过度设计架构**：在简单 CRUD 模块强塞复杂的六边形架构或 CQRS；
7. **全库格式化污染 Git 提交**：改动单接口却顺带格式化全库几十个文件，破坏 `git blame`；
8. **随意拉升依赖版本**：无端升级主要依赖库，破坏项目既有兼容性。

---

# 28. 交付完成定义 (Definition of Done - DoD)

在宣布任务完成前，对照以下清单自检：

- [ ] 已透彻理解项目既有分层架构与上下文
- [ ] 严格遵守单一职责，单文件未超过 500 行，一个用例对应一个 Logic
- [ ] 涉及接口变动已正确更新 `.api` / `.proto` 源契约
- [ ] 涉及生成代码已通过 Makefile 受控重新生成，未手改生成文件
- [ ] 核心业务逻辑收敛于 Logic 层，未在 Handler 写业务
- [ ] 枚举符合三层转换标准，无魔法数字或硬编码字符串判断
- [ ] 数据库 CRUD 与缓存实现正确，无 SQL 注入与全表更新风险
- [ ] 分布式锁使用随机 UUID + Lua 脚本原子释放
- [ ] 错误处理遵循 5 大场景规范，跨层 `%w` 包装，无 `_` 忽略
- [ ] 认证鉴权与租户数据隔离安全受检
- [ ] 单元测试已编写或更新，覆盖核心与边界用例
- [ ] `gofmt` 已执行，`go test ./...` 运行通过
- [ ] `git diff` 严格复查无误，无无关文件混入，无明文密钥泄露

---

# 29. 决策优先级仲裁 (Priority Rules)

面临方案分歧时，严格按下述优先级裁决：

```text
1. 用户的明确指令与要求 (User's Explicit Requirements)
   ↓
2. 当前项目既有的具体架构规范与已有约定 (Existing Project Architecture & Conventions)
   ↓
3. go-zero 官方标准范式 (go-zero Conventions)
   ↓
4. Go 语言惯用语法 (Idiomatic Go)
   ↓
5. 通用软件工程最佳实践 (General Engineering Best Practices)
```

---

# 30. 黄金准则 (The Golden Rule)

针对 go-zero 工程开发，永恒的核心法则只有一条：

> **既有项目代码是第一事实来源 (The existing project is the primary source of truth)。**

- 动手造轮子前，先在已有项目中找示例；
- 修改生成的代码前，先找到它的定义源头；
- 引入第三方库前，先检索仓库既有依赖；
- 更改接口契约前，先评估所有调用方影响；
- 始终保持改动：**最小精准、语义明确、测试覆盖、稳定兼容！**
