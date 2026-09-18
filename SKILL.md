---
name: go-zero-development
description: >-
  Go 语言与 go-zero 高性能微服务架构开发规范与工程技能。
  涵盖 goctl 代码生成边界、API / gRPC RPC 契约设计、Handler / Logic / ServiceContext / Model 四层分层职责、
  GORM / MySQL / MongoDB / Redis 数据库与缓存最佳实践、Goroutine Panic 拦截与并发安全红线、
  多租户隔离、业务错误包装、测试验证、Docker 与 GitLab CI/CD 规范。
---

# Go-Zero 开发技能规范 (Go-Zero Development Skill)

## 概述 (Overview)

本技能旨在为 AI 代理（AI Coding Agents）以及研发工程师在基于 **go-zero** 框架进行 Go 语言微服务系统的**新增功能、代码审查、Bug 排查排障、架构重构或接口扩展**时，提供权威、严密且符合生产环境标准的工程规范与行动指引。

### 核心指导原则

1. **遵循既有架构**：优先遵循并对齐当前项目已建立的代码组织架构与命名约定。
2. **坚持 Go 与 go-zero 惯用模式**：采用地道（idiomatic）的 Go 语言语法与 go-zero 官方推荐的设计模式。
3. **严守代码生成边界**：尊重 `goctl` 的代码生成体系，坚决杜绝手动随意篡改由工具自动生成的代码。
4. **最小化非必要变更**：恪守原子化修改原则，坚决不做无关重构与大面积格式化。
5. **保持向后兼容**：除非用户明确下达破坏性变更指令，否则必须确保接口与数据结构的向后兼容性。
6. **优先复用既有抽象**：在引入新的库、工具函数或自定义抽象之前，必须先在现有代码库中搜索复用。
7. **业务逻辑高可测易维护**：确保 Logic 层业务代码解耦、依赖倒置，易于进行单元测试与表格驱动测试。
8. **拒绝盲目引入依赖**：严禁在无充分业务技术依据的情况下私自引入第三方框架或冗余依赖包。

### 适用技术栈

- **Language**: Go (1.20+)
- **Core Framework**: go-zero, goctl
- **Protocols**: RESTful HTTP API, gRPC / zrpc, Protocol Buffers (v3)
- **Data Persistence**: GORM / GORM Gen, goctl model (sqlc), MySQL, MongoDB
- **Caching & KV**: Redis
- **Infra & DevOps**: Docker (Multi-stage build), GitLab CI/CD

---

# 1. 通用规则 (General Rules)

## 1.1 修改前先调研 (Inspect Before Modifying)

在动手修改或创建任何代码之前，必须先按以下顺序执行调研：

1. **审视仓库结构**：梳理各微服务的目录划分方式（单体大仓库 Monorepo 还是独立服务拆分）。
2. **查阅项目规范**：阅读仓库中的 `README.md`、`AGENTS.md`、`CONTRIBUTING.md` 或 `project_wiki/` 技术文档。
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

### 典型 go-zero 项目工程布局
```text
.
├── api/                           # HTTP API 服务根目录
│   ├── etc/                       # 运行配置文件 (yaml)
│   ├── internal/
│   │   ├── config/                # 配置结构体定义
│   │   ├── handler/               # HTTP Handler 路由控制器 (goctl 生成，禁写业务)
│   │   ├── logic/                 # 核心业务逻辑编排 (业务核心)
│   │   ├── svc/                   # 服务依赖注入上下文 (ServiceContext)
│   │   └── types/                 # 请求/响应 DTO 结构体 (goctl 生成)
│   └── user.api                   # API 接口 DSL 契约定义
│
├── rpc/                           # gRPC 微服务根目录
│   ├── etc/                       # RPC 运行配置
│   ├── internal/
│   │   ├── config/                # RPC 配置结构体
│   │   ├── logic/                 # RPC 业务实现
│   │   ├── server/                # gRPC Server 实现 (goctl 生成)
│   │   ├── svc/                   # RPC 服务上下文
│   │   └── model/                 # RPC 私有模型 (若有)
│   └── user.proto                 # Protobuf 协议契约
│
├── model/                         # 数据访问层 (GORM Gen 或 goctl model)
├── common/                        # 跨模块通用常量、工具函数、自定义错误码
├── pkg/                           # 公共库与基础支撑组件
├── scripts/                       # 自动化构建、生成、迁移脚本
├── Dockerfile                     # 容器镜像构建文件
└── go.mod                         # Go 模块定义
```

> **注意**：各公司实际工程可能会采用单体多服务（Monorepo）或拆分子目录，必须以实际仓库目录为准。

---

# 3. go-zero API 开发规范

## 3.1 API 定义为单一事实来源 (API Definitions Are the Source of Truth)

在 go-zero 中，所有 HTTP 接口的输入、输出、路径及路由分组均由 `.api` 文件唯一定义。

- **正确的开发顺序**：
  1. 修改或新增 `.api` 文件中的数据结构与路由声明；
  2. 使用 `goctl api go` 命令重新生成代码；
  3. 填充并编写新生成在 `internal/logic/` 中的业务逻辑。
- **红线约束**：
  - **严禁手动编辑由 goctl 生成的文件**（如 `internal/handler/*`、`internal/types/types.go`）。
  - 若结构体字段需要调整，必须修改源 `.api` 文件后重新执行代码生成。

```go
// user.api 示例
type UserRequest {
    Name string `json:"name"`
}

type UserResponse {
    UserID int64 `json:"user_id,string"`
}

service user-api {
    @handler CreateUser
    post /users (UserRequest) returns (UserResponse)
}
```

## 3.2 JSON 序列化 Tag 与精度保护

必须深刻理解整型字段是否需要携带 `,string` 修饰符：

```go
// 方式 A：普通整型输出
UserID int64 `json:"user_id"`

// 方式 B：字符串形式输出（防前端 JS 精度截断）
UserID int64 `json:"user_id,string"`
```

- **业务背景**：JavaScript 中的 `Number.MAX_SAFE_INTEGER` 为 $2^{53}-1$（9007199254740991）。若项目使用雪花算法（Snowflake ID）生成 64 位整数 ID，前端接收会发生精度丢失（低位被截断成 0）。
- **规范**：涉及 64 位雪花 ID 或超大整数主键，必须使用 `json:"id,string"`。
- **防踩坑**：**严禁在已有稳定接口上随意添加或移除 `,string`**，此类变更会直接破坏移动端或前端反序列化，必须与前后端调用方严格对齐。

## 3.3 请求参数校验与分工 (Request Validation)

明确两层校验的职责划分：

```text
Handler 层 (反序列化与基础语法校验)
  ↓ 依靠 httpx.Parse 及 struct 校验 tag (optional, range, validate 等)
Logic 层 (业务语义校验)
  ↓ 校验数据库唯一性、业务状态流转有效性、账号配额等
```

```go
type CreateUserRequest {
    Name string `json:"name"`
    Age  int    `json:"age,optional"` // optional 标记非必填字段
}
```

- 严禁在 Handler 与 Logic 中做无意义的重复校验。
- 业务校验失败时返回明确的业务错误，不要笼统报 500。

---

# 4. Handler 控制层规范 (Handler Rules)

Handler 必须保持为“薄控制器”（Thin Handler），其唯一职责是协议适配与流程编排：

### 标准 Handler 范式
```go
func CreateUserHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req types.CreateUserRequest
        // 1. 参数解析与反序列化
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
- **绝对禁止在 Handler 中编写业务逻辑代码**。
- **绝对禁止在 Handler 中直接查询数据库或操作 Redis**。
- **绝对禁止在 Handler 中启动复杂数据库事务**。
- 统一由 `httpx.OkJsonCtx` 与 `httpx.ErrorCtx` 进行响应返回，确保 Trace ID 正确串联。

---

# 5. Logic 业务层规范 (Logic Layer)

Logic 层是所有核心业务规则的聚合实现中心。

### Logic 方法标准职责
1. 校验复杂业务规则（如资格前置检查、状态机约束）。
2. 调用持久化 Model 层或下游 RPC 微服务。
3. 执行纯内存业务运算与数据转换。
4. 处理底层错误并包装为带业务语义的错误返回。
5. 构造并返回响应 DTO。

### 基础设施隔离红线
- **严禁在 Logic 内部动态初始化基础设施连接**。
- **禁止在 Logic 函数内部调用 `gorm.Open` 或创建 Redis 客户端连接**：
  ```go
  // 错误示范：严禁在 Logic 中动态建立连接！
  func (l *CreateUserLogic) CreateUser(req *types.CreateUserRequest) error {
      db, _ := gorm.Open(mysql.Open(...)) // 严重错误！连接泄露且脱离框架管理
      ...
  }
  ```
- 所有数据库、缓存和第三方客户端必须统一从 `l.svcCtx`（`ServiceContext`）中获取。

---

# 6. ServiceContext 与依赖注入规范 (ServiceContext & DI)

go-zero 采用 `ServiceContext` 作为微服务的依赖注入（Dependency Injection）中心。

```go
type ServiceContext struct {
    Config    config.Config
    UserModel model.UserModel
    Redis     *redis.Redis
    UserRpc   userclient.User
    DB        *gorm.DB
}
```

- **单例资源注入**：数据库连接池（`*gorm.DB` / `sqlx.SqlConn`）、Redis 客户端、下游 RPC Client 必须在 `NewServiceContext` 初始化时统一完成创建，全服务共享。
- **严禁引入全局可变变量**：禁止定义 `var GlobalDB *gorm.DB`，全局变量会破坏单元测试 Mock、导致并发竞争与状态污染。
- **消费方式**：在 Logic 中统一通过 `l.svcCtx.XXX` 引用。

---

# 7. goctl 代码生成规范 (goctl Guidelines)

`goctl` 是 go-zero 生态最核心的效率工具，但也是最容易被 AI 滥用的利刃。

### 生成源头矩阵
| 目标变更 | 事实来源 (Source of Truth) | 生成命令范式 |
| :--- | :--- | :--- |
| HTTP API / DTO | `*.api` | `goctl api go -api *.api -dir .` |
| RPC 服务 / 存根 | `*.proto` | `goctl rpc protoc *.proto --go_out=. --go-grpc_out=. --zrpc_out=.` |
| 数据库 Model | DDL SQL / 数据表 | `goctl model mysql ddl -src *.sql -dir ./model -c` |

### 严禁盲目执行全量重新生成 (Never Blindly Regenerate)
执行任何代码生成之前，必须遵循安全 5 步流程：
1. **检查版本一致性**：运行 `goctl --version`，确保本地工具版本与项目既有版本一致。
2. **检查 Git 工作区状态**：执行 `git status --short`，确保当前无未提交的脏改动。
3. **确认目标变更文件**：明确了解即将被覆盖的文件清单。
4. **优先使用 Makefile/脚本**：如果项目根目录有 `Makefile`（如 `make gen-api` / `make gen-rpc`），**必须强制优先调用 Makefile 指令**。
5. **审阅代码差异**：生成完毕后立刻运行 `git diff`，严格确认没有破坏或误抹除已有手工逻辑（如 Logic 实现）。

---

# 8. 生成代码边界控制 (Generated Code Boundaries)

由 `goctl` 自动生成的代码文件在日常开发中**一律视为只读（Read-Only）**：

- **典型只读目录/文件**：
  - `internal/handler/`
  - `internal/types/types.go`
  - `internal/server/`
  - `model/*_gen.go`
  - `*client/*.go`
- **处理原则**：如果生成的代码存在字段缺失、Tag 不正确或路由错误：
  - **严禁直接在生成文件中打补丁**（下次执行生成时会被无情覆盖）。
  - 必须溯源到根源：修改 `.api`、`.proto`、数据表 DDL、模板或生成脚本，重新生成。
  - **唯一例外**：项目官方或团队规则中显式声明该目录不重新生成且允许手工维护。

---

# 9. RPC / zrpc 跨服务通信规范

在基于 zrpc 构建的微服务间交互中，必须遵循 Protocol Buffers 契约边界：

```text
API Logic
  ↓
l.svcCtx.UserRpc.GetUser(l.ctx, &user.IdRequest{Id: 1001})
  ↓ [zrpc 负载均衡 / 熔断 / 超时控制]
User RPC Service
```

- **严禁越俎代庖直连异构服务数据库**：若某类数据属于用户微服务，订单服务**严禁直接连用户服务的 MySQL 表查询**，必须通过 `UserRpc` 客户端经由 RPC 接口访问。

### Proto 契约变更原则
1. **向后兼容**：新增字段使用全新的 tag 编号，严禁修改已有字段的含义与类型。
2. **禁止复用废弃编号**：被删除的 protobuf 字段编号严禁给新字段重复使用，必须声明 `reserved`：
   ```protobuf
   message UserDetail {
       reserved 5, 8 to 11;
       string name = 1;
       int64 id = 2;
   }
   ```
3. **变更标准流**：修改 `.proto` → 检查兼容性 → 运行生成命令 → 更新服务端 Logic → 更新客户端调用代码 → 运行单测验证。

---

# 10. 数据库与持久化规范 - MySQL

1. **统一技术选型**：严格使用项目既有的持久化抽象（如 GORM 或 go-zero 内置 sqlc/model），**严禁为了新需求在同一模块中擅自引入第二套 ORM**。
2. **禁止混合使用**：严禁在同一业务 Logic 中混用 GORM、sqlx、database/sql 和手写原生 SQL。
3. **注释规范**：所有数据库表结构定义（DDL）以及 Model 结构体必须包含详尽的中文业务含义注释。

---

# 11. GORM / GORM Gen 工程规范

### 11.1 防 SQL 注入与参数化查询
- 必须使用参数化占位符：
  ```go
  // 正确：参数化查询
  db.WithContext(ctx).Where("user_id = ?", userID).First(&user)

  // 严禁：字符串拼接防注入漏洞！
  db.Where(fmt.Sprintf("user_id = %d", userID)) // 绝对禁止！
  ```

### 11.2 零值陷阱与精准更新
- 使用 Struct 更新时，GORM 默认会**忽略所有零值字段**（如 `false`, `0`, `""`）：
  ```go
  // 若 Age 为 0 或 Status 为 false，此写法不会更新对应数据库字段！
  db.Model(&user).Updates(User{Age: 0, Status: false})

  // 正确方案：明确指定 Select 字段或使用 map
  db.Model(&user).Select("Age", "Status").Updates(User{Age: 0, Status: false})
  // 或
  db.Model(&user).Updates(map[string]any{"age": 0, "status": false})
  ```

### 11.3 杜绝无条件全表更新或删除
执行 `Delete` 或 `Update` 必须确保带有明确的 `Where` 条件，防范灾难性全表清空。

### 11.4 事务标准范式
多步骤写操作且需要原子性保障的，必须使用闭包事务：
```go
err := db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return fmt.Errorf("create order failed: %w", err)
    }
    if err := tx.Model(&inventory).Where("item_id = ? AND stock >= ?", itemID, num).
        UpdateColumn("stock", gorm.Expr("stock - ?", num)).Error; err != nil {
        return fmt.Errorf("deduct stock failed: %w", err)
    }
    return nil
})
```
- **严禁**将单次读操作包装在事务中（无意义消耗 DB 连接池与锁资源）。

---

# 12. MongoDB 规范

- **必须传递上下文**：所有 MongoDB 操作首个参数必须显式传递 `ctx`：
  ```go
  err := collection.FindOne(ctx, bson.M{"_id": objectID}).Decode(&result)
  ```
- **区分“数据不存在”与“系统故障”**：对于 `mongo.ErrNoDocuments`，应根据业务语义转译为明确的业务 NotFound 状态，而不是笼统当成 500 系统异常抛出。

---

# 13. Redis 缓存规范

1. **依赖统一注入**：通过 `ServiceContext` 注入的 `*redis.Redis` 或连接池单例操作，严禁在业务层每次建立连接。
2. **缓存四要素明确**：
   - **Key 命名规范**：带业务模块前缀与版本号，如 `user:profile:v1:{user_id}`。
   - **TTL 超时机制**：所有写入缓存的数据必须显式设置过期时间（TTL），严禁无脑写入永久 Key。
   - **序列化效率**：对齐项目既有序列化方案（JSON / Protobuf）。
   - **双写与失效**：写操作后及时主动失效或更新缓存。
3. **严禁过度缓存**：不要因为 Redis 存在而对所有只读一次的低频数据加缓存，徒增一致性维护成本。

---

# 14. 错误处理与分层包装 (Error Handling)

### 14.1 错误包装与上下文追溯
- 跨层传递错误时，使用 `%w` 动词包装原始错误，保留调用栈链路：
  ```go
  if err != nil {
      return nil, fmt.Errorf("query user detail [id=%d]: %w", req.ID, err)
  }
  ```
- **核心红线**：
  - **严禁使用 `_` 匿名忽略 error**。
  - **业务层与 Logic 层中坚决禁止主动调用 `panic()`**。
  - 避免无意义的连续包裹（避免出现 `failed: failed: failed` 式废话日志）。

### 14.2 业务错误码与系统边界
- 遵循统一错误码体系，严格区分五类错误：
  1. **参数校验错误**（Validation Error - 400）
  2. **业务逻辑错误**（Business Error - 422 / 自定义业务 Code）
  3. **数据库故障**（DB Infrastructure Error）
  4. **远程 RPC 超时/失败**（RPC Communication Error）
  5. **未知系统崩溃**（Internal Server Error - 500）
- **安全红线**：严禁把底层原始 SQL 报错信息（如 `Duplicate entry ... for key ...`）直接裸露返回给前端用户。

---

# 15. Context 上下文传递规范 (Context Propagation)

1. **显式第一参数传递**：所有涉及网络 I/O、RPC 调用、数据库操作的方法，必须将 `ctx context.Context` 作为方法的第一个参数：
   ```go
   func (m *defaultUserModel) FindOne(ctx context.Context, id int64) (*User, error)
   ```
2. **严禁上下文断流**：在请求处理链路中，**严禁无故使用 `context.Background()`** 替代来自上游的 `r.Context()` 或 `l.ctx`，否则链路追踪 Trace ID 和超时取消将彻底失效。
3. **禁止长存结构体**：禁止将 `context.Context` 作为长期存活结构体的持久字段。
4. **禁止作为传参口袋**：禁止使用 `context.WithValue` 传递核心业务参数，Context 仅用于传递元数据（TraceID、AuthClaims、TenantID、Deadline）。

---

# 16. 并发控制与结构化并发 (Concurrency)

在 Go 中启动并发协程前，必须回答 6 个问题：
1. 是否真的能显著降低接口延迟？
2. 是 I/O 密集型还是 CPU 密集型？
3. 子任务失败时如何聚合与上报错误？
4. 父任务超时取消时子任务能否感知并优雅退出？
5. 是否存在 Goroutine 泄露隐患？
6. 共享状态是否受到充分的并发保护？

- **推荐模式**：使用 `golang.org/x/sync/errgroup` 实现结构化并发。
- **验证红线**：所有包含并发逻辑的代码修改，必须运行 `go test -race ./...` 进行竞态检测。

---

# 17. Goroutine 并发安全红线 (Goroutine Safety)

### 核心红线 1：Goroutine Panic 拦截
所有新启动的后台 Goroutine，**必须在函数首行使用 `defer-recover` 拦截 `panic`**，严防协程崩溃击垮整个后端服务进程：

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

### 核心红线 2：循环变量闭包捕获陷阱
注意 Go 循环变量捕获问题，必须通过形参显式传入：
```go
for _, item := range list {
    item := item // Go 1.22 之前版本必须显式重赋值；或作为参数传入协程
    go func(val Data) {
        process(val)
    }(item)
}
```

---

# 18. API 兼容性与版本治理 (API Compatibility)

在修改已有接口时，必须先评估上下游影响：
- **评估项**：调用方是前端 Web、移动端 App、内部微服务还是第三方开放平台？
- **破坏性变更禁令**：
  - 严禁擅自修改已发布的 JSON 字段名、大小写。
  - 严禁随意变更字段类型（如 `int` 改为 `string`）。
  - 严禁随意更改 HTTP Method（如 `GET` 改为 `POST`）或 URL 路径。
- **推荐策略**：字段扩展采用“只增不改”，废弃字段保留过渡期。

---

# 19. 日志打印与安全脱敏 (Logging)

1. **统一使用 go-zero logx**：使用 `logx.WithContext(ctx)` 记录日志，确保每条日志都携带 OpenTelemetry 分布式链路 TraceID。
2. **严禁日志裸印敏感凭证**：
   - 严禁打印密码、明文 Token、JWT Secret、银行卡号、手机号明文、API Key。
3. **禁止盲目 Dumping 大对象**：禁止在高频接口的生产日志中无脑使用 `logx.Infof("req: %+v", req)` 打印包含完整大数组的对象，防止磁盘 I/O 打满和日志成本失控。

---

# 20. 配置管理规范 (Configuration)

1. **严禁硬编码**：数据库连接串、Redis 密码、RPC 监听地址、JWT 密钥必须定义在 `config.Config` 中并通过 YAML 配置与环境变量读取。
2. **环境隔离**：生产、测试、开发环境配置分离。
3. **安全红线**：**绝对禁止将包含真实密钥、生产密码的配置文件提交到 Git 仓库**。

---

# 21. 认证与授权边界 (Authentication & Authorization)

- **认证 (Authentication - Who are you)**：验证调用方身份（JWT 校验、Token 解密）。
- **授权 (Authorization - What are you allowed to do)**：验证是否有权操作该资源（RBAC、资源归属检查）。
- **安全红线**：
  - **坚决禁止盲目信任客户端请求体中传入的 `user_id`**！
  - 当前操作用户 ID 必须从 Context 中的 JWT Claims 安全提取；只有在已确认其具备管理员权限时，才允许操作指定目标 `user_id` 的数据。

---

# 22. 多租户系统隔离规范 (Multi-Tenant Systems)

对于具备多租户特性的系统：
1. **租户凭证自闭环**：租户身份（`tenant_id`）必须从认证 Context 中提取，禁止直接信任前端 URL Query 或 Body 中可被篡改的租户字段。
2. **查询条件必带租户过滤**：所有针对业务表的增删改查 SQL，必须无条件携带 `tenant_id = ?` 约束：
   ```sql
   SELECT * FROM orders WHERE id = ? AND tenant_id = ?;
   ```
3. **防越权**：漏加租户过滤是严重的越权数据泄漏漏洞。

---

# 23. 事务与分布式一致性 (Transactions)

1. **明确事务边界**：只有多步强关联的数据库写入才需要开事务。
2. **分清分布式边界**：**MySQL 本地事务管不到 Redis 与下游 RPC 调用**！
   - 严禁在本地数据库事务未提交前去调用外部耗时较长的 RPC 接口（会拖长 DB 锁持有时间导致连接池耗尽）。
3. **最终一致性**：跨服务的一致性应基于可靠消息队列、Saga 或 TCC 模式，明确补偿策略。

---

# 24. 性能调优原则 (Performance)

1. **基于数据与 Profiling 说话**：拒绝盲目凭直觉“负优化”。
2. **常用分析工具**：
   ```bash
   go test -bench . -benchmem  # 基准性能测试
   go tool pprof              # CPU / Memory 性能分析
   go test -race ./...         # 竞态检测
   ```
3. **数据库性能优先看执行计划**：
   ```sql
   EXPLAIN SELECT ...
   ```
   核心关注：命中索引（`key`）、扫描行数（`rows`）、是否出现 `Using filesort` 或 `Using temporary`。

---

# 25. 数据库查询守则 (Database Query Rules)

编写 SQL 或 ORM 查询前，必须完成以下自检：
1. 目标表的数据量级是多少？
2. 查询条件列是否已建立合适索引？联合索引是否符合最左前缀原则？
3. 该查询在生产环境的峰值 QPS 是多少？
4. 是否强制要求分页？
5. **高频业务路径上严禁无脑 `SELECT *`**，必须使用 `Select("id", "status", "created_at")` 仅投影必要字段。

---

# 26. 分页设计与防深度分页 (Pagination)

1. **常规轻量分页**：数据量小于数万行时，可使用标准 `LIMIT ? OFFSET ?`。
2. **深度分页防劣化**：对于数百万级的大表，严禁直接使用大 Offset 分页（如 `OFFSET 1000000` 会导致全表扫描大量回表）。应采用游标/主键 Keyset 分页：
   ```sql
   WHERE id > ? ORDER BY id ASC LIMIT ?
   ```

---

# 27. 单元测试规范 (Testing)

所有关键业务逻辑与边界计算必须编写单元测试：
- **优先采用 Go 官方推荐的表格驱动测试（Table-Driven Tests）**：
  ```go
  func TestCreateUser(t *testing.T) {
      tests := []struct {
          name    string
          input   Request
          wantErr bool
      }{
          {name: "正常创建用户", input: Request{Name: "Alice"}, wantErr: false},
          {name: "缺少用户名", input: Request{Name: ""}, wantErr: true},
      }
      for _, tt := range tests {
          t.Run(tt.name, func(t *testing.T) {
              // 执行测试逻辑并断言
          })
      }
  }
  ```
- **核心覆盖场景**：正常流、临界边界条件、非法输入校验、空指针防御、DB/RPC 异常降级。

---

# 28. 质量检查与测试命令集 (Testing Commands)

在交付代码前，必须依次运行以下命令验证代码质量：

```bash
# 1. 运行所有单元测试
go test ./...

# 2. 运行竞态检查 (涉及并发改动时必跑)
go test -race ./...

# 3. 规范化代码格式
gofmt -w .

# 4. 执行官方静态代码分析
go vet ./...

# 5. 执行团队定制 Lint (若项目配置了 golangci-lint)
golangci-lint run
```

---

# 29. 安全重构守则 (Refactoring)

重构代码时必须遵循小步迭代原则：
1. 明确现有逻辑的基线行为并确保已有测试覆盖。
2. 实施最小粒度的结构重构。
3. 运行完整测试套件，核对预期行为。
4. 审查 `git diff`，确认无额外副作用。
5. **严禁混为一谈**：禁止在同一个需求交付中同时塞入“业务新需求 + 翻天覆地的大重构 + 框架大版本升级 + 全库格式化”。

---

# 30. 依赖包管理规范 (Dependency Management)

1. **首选标准库**：能用 Go 标准库优雅解决的问题，不要引入第三方库。
2. **优先复用既有库**：引入新包前先查看 `go.mod` 是否已有类似库。
3. **考察库的健康度**：必须选择社区活跃、Star 数高、近期有维护的成熟库。
4. **依赖变更后**：
   ```bash
   go mod tidy
   git diff go.mod go.sum
   ```
   严格检查并确认没有多余或恶意的间接依赖引入。

---

# 31. Go 语言版本适配 (Go Version)

- 在编写代码或引入语言特性前，**必须首先查阅项目根目录下的 `go.mod` 中声明的 Go 版本**。
- 严禁使用高于当前项目 `go.mod` 声明版本的语法特性（例如在 Go 1.20 项目中直接使用 Go 1.22 的循环变量新特性，会导致低版本编译机构建失败）。

---

# 32. Dockerfile 容器化构建规范

编写或调整 Dockerfile 时必须满足以下生产标准：
- **采用多阶段构建（Multi-Stage Build）**，严格控制最终镜像体积与安全面。
- **静态编译设置**：纯 Go 服务构建时显式设置 `CGO_ENABLED=0`。
- **运行时环境**：携带时区数据（`tzdata`）与根证书（`ca-certificates`）。

```dockerfile
# 构建阶段
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/service ./service.go

# 运行阶段
FROM alpine:latest
RUN apk --no-cache add ca-certificates tzdata
ENV TZ=Asia/Shanghai
WORKDIR /app
COPY --from=builder /app/service /app/service
COPY --from=builder /app/etc /app/etc
ENTRYPOINT ["/app/service", "-f", "etc/service.yaml"]
```

---

# 33. CI/CD 流水线规范 (GitLab CI / GitHub Actions)

- **流水线标准阶段**：
  ```text
  build (编译检查) → test (单元测试/竞态) → lint (代码质量) → package (镜像构建) → deploy (环境发布)
  ```
- **配置一致性**：严格基于项目既有 `.gitlab-ci.yml` 进行增量维护，禁止任意推翻重构既有 CI 流程。
- **私有仓库鉴权**：企业内部模块需在 CI 中正确配置凭据并设置 `GOPRIVATE`。
- **安全红线**：禁止在 CI 配置文件中直接写入明文密码或生产 Token。

---

# 34. 私有 Go 模块支持 (Private Go Modules)

对于组织内部私有 Go 仓库（如 `gitlab.company.com/backend/pkg`）：
- 合理配置 Go 环境变量：
  ```bash
  go env -w GOPRIVATE="gitlab.company.com/*"
  ```
- 遵循企业配置的统一私有 GOPROXY 与认证方案，切勿在不清楚影响的情况下全局关闭 `GONOSUMDB`。

---

# 35. 规范代码风格 (Code Style)

1. **卫语句（Guard Clauses）优先**：尽早返回，降低代码圈复杂度与嵌套层级。
   ```go
   // 推荐：提前返回
   if err != nil {
       return nil, err
   }

   // 避免：深层嵌套
   if err == nil {
       if val != "" {
           ...
       }
   }
   ```
2. **小函数原则**：保持单个函数职责单一，避免数百行的巨型函数。
3. **拒绝接口滥用**：
   - 只有在存在多个实现类、需要跨系统 Mock 测试解耦、或存在明确架构隔离边界时才定义 `interface`。
   - **禁止为了写接口而写单实现接口**（不要把 Java 的套路生搬硬套到 Go 中）。

---

# 36. AI 常见错误与避坑指南 (Common AI Mistakes to Avoid)

AI 代理在开发 go-zero 体系代码时，极易犯下以下致命错误，必须坚决杜绝：

1. **擅自篡改底层架构**：把 go-zero 的路由与 Handler 强行改成 Gin、Fiber、Echo 或 Kratos 风格。
2. **直接手工修改生成代码**：手动修改 `internal/handler/` 或 `*_gen.go`，导致下次 `goctl` 生成被全面覆盖。
3. **滥建冗余接口（Over-interfacing）**：无端为每个 Logic 凭空生成 `type UserService interface`。
4. **随处定义全局数据库客户端**：搞出 `var DB *gorm.DB` 全局单例，无视 `ServiceContext` 的依赖注入体系。
5. **对既有通用工具视而不见**：重新造轮子实现分页结构体、统一响应体或加解密工具，无视项目已有的公共包。
6. **过度设计架构（Overengineering）**：在简单的 CRUD 模块上强塞 DDD、CQRS、事件溯源或复杂的六边形架构。
7. **全库格式化污染 Git 提交**：改动一个接口却顺带把全库几十个文件全部 `gofmt` 了一遍，导致 `git blame` 历史记录彻底失效。
8. **随意拉升依赖主版本**：因为最新版可用就私自把 `go-zero` 从 `v1.5.x` 升到 `v1.7.x`，触发大面积 API 破坏。

---

# 37. 代码检索与定位策略 (Search Strategy)

在动手编写代码之前，先在终端或编辑器中针对业务概念执行全局精准搜索：

```bash
# 检索既有请求/响应结构
rg "type .*Request" .

# 检索现有 Logic 构造范式
rg "New.*Logic" .

# 检索 ServiceContext 依赖注入
rg "ServiceContext" .

# 检索统一错误码定义
rg "Err.*" .

# 检索事务使用范式
rg "Transaction" .

# 检索类似业务 Model 方法
rg "FindBy" .
```

> **检索启发**：如果要开发 `CreateOrder` 接口，请先搜索项目中已有的 `Create`、`OrderModel`、`orderRpc`、`ErrOrder`、`order_id`，100% 对齐现有命名风格。

---

# 38. 变更规划工作流 (Change Planning)

对于任何非平凡的接口开发或重构任务，必须遵循规范的思考与规划链路：

```text
1. 修改/新增 .api 或 .proto 契约文件
2. 运行 goctl 生成对应 Handler / Server / Types 框架
3. 在 ServiceContext 中注入所需的数据持久层或 RPC 客户端
4. 编写 Logic 层业务规则实现并串联 Model
5. 编写单元测试用例 (Table-Driven Tests)
6. 运行 gofmt、go vet 与 go test 保证质量
7. 审阅本地 git diff，确保改动精准无污染
```

---

# 39. Git 安全守则 (Git Safety)

1. **修改前基线检查**：运行 `git status --short`，明确已有变更。
2. **修改后差异复审**：运行 `git diff --stat` 和 `git diff`，仔细检查每一行变动。
3. **禁止破坏性指令**：**绝对禁止在未经用户明确要求的情况下运行 `git reset --hard` 或 `git clean -fd`**！严禁私自丢弃用户工作区中的未暂存改动。

---

# 40. 交付完成定义 (Definition of Done - DoD)

在向用户宣布任务完成之前，逐项对照以下清单进行自检：

- [ ] 已透彻理解项目既有分层架构与上下文
- [ ] 已完整复用项目现有的实现范式与工具库
- [ ] 涉及接口变动已正确更新 `.api` / `.proto` 源文件
- [ ] 涉及生成代码已受控重新生成，未手改生成文件
- [ ] 核心业务逻辑在 Logic 层编写完毕
- [ ] 数据库 CRUD 与缓存逻辑实现正确、无 SQL 注入隐患
- [ ] 跨服务 RPC 客户端调用安全并处理超时
- [ ] 错误处理规范完整，跨层使用 `%w` 包装，无 `_` 忽略
- [ ] 认证鉴权与租户数据隔离安全受检
- [ ] 单元测试已编写或更新，覆盖核心与边界用例
- [ ] 已执行 `gofmt` 保持格式美观统一
- [ ] `go test ./...` 运行无报错
- [ ] `git diff` 严格复查无误，无无关文件混入
- [ ] 无任何硬编码密钥或密码泄露风险

---

# 41. Agent 标准行为流 (Agent Workflow)

AI 代理在接收到 go-zero 相关开发任务时，必须严格执行以下 7 步标准化闭环：

### 步骤 1 — 理解调研 (Understand)
检查 `go.mod`、项目目录树、`.api` / `.proto`、`ServiceContext`、类似业务的 Logic/Model 以及既有单元测试。

### 步骤 2 — 架构规划 (Plan)
确定实现需求所需的最小文件集，梳理数据流转与依赖关系。

### 步骤 3 — 遵循实现 (Implement)
严格沿用既有设计模式，将业务逻辑收敛于 Logic 层。

### 步骤 4 — 受控生成 (Generate)
若修改了契约文件，使用项目既定指令（Makefile/goctl）受控生成代码。

### 步骤 5 — 质量验证 (Verify)
运行 `gofmt`、`go vet` 以及 `go test ./...`（必要时加 `-race`），确保编译与测试全绿。

### 步骤 6 — 差异审查 (Review)
运行 `git diff` 与 `git status`，严格杜绝多余修改与生成污染。

### 步骤 7 — 总结汇报 (Report)
结构化向用户汇报：修改了哪些文件、完成了哪些逻辑、执行了何种代码生成、验证了哪些测试，并提示潜在的注意事项。

---

# 42. 决策优先级仲裁 (Priority Rules)

当在编码过程中面临多个方案分歧时，必须严格按下述优先级裁决：

```text
1. 用户的明确指令与要求 (User's Explicit Requirements)
   ↓
2. 当前项目既有的具体架构规范 (Existing Project Architecture)
   ↓
3. 当前项目的代码习惯与命名约定 (Existing Project Conventions)
   ↓
4. go-zero 官方标准范式 (go-zero Conventions)
   ↓
5. Go 语言惯用语法 (Idiomatic Go)
   ↓
6. 通用软件工程最佳实践 (General Engineering Best Practices)
```

**严禁**以宽泛的“通用最佳实践”为由，强行推翻当前项目已长期沉淀并稳定运行的特定约定！

---

# 43. 黄金准则 (The Golden Rule)

针对 go-zero 工程开发，永恒的核心法则只有一条：

> **既有项目代码是第一事实来源 (The existing project is the primary source of truth)。**

- 动手造轮子前，先在已有项目中找示例；
- 修改生成的代码前，先找到它的定义源头；
- 引入第三方库前，先检索仓库既有依赖；
- 更改接口契约前，先评估所有调用方影响；
- 实施性能优化前，先用工具分析真实瓶颈；
- 宣布完工交付前，认真复查每一行 `git diff`。

**始终保持改动：最小精准、语义明确、测试覆盖、稳定兼容！**
