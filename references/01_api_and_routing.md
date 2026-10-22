# API 契约、路由设计与代码生成参考手册

## 1. API 契约与路由设计规范

### 1.1 契约驱动与单一真理源 (SSOT)
- **开发顺序**：修改 `.api` 契约 ➔ `make <target>` 标准化生成 ➔ 编写 Logic 业务实现；
- **只读铁律**：严禁手动编辑由 goctl 生成的 `internal/handler/`、`internal/types/`、`internal/routes.go`，改动必须回源到 `.api` 重新生成。

### 1.2 路由分层与路径命名
- **外部公网路由**：`/web-api/v1/{module}/{action}`（面向终端、前端应用）；
- **内部集群路由**：`/inner-api/v1/{service}/{module}/{action}`（微服务专用，严禁外网暴露）；
- **命名规范**：路径短语统一使用小写中划线连接 `/xxxx-xxxx/`（如 `/user-group/detail`），禁下划线与驼峰。

### 1.3 核心动作动词规范 (Action Verbs)
| 动词 | 语义 | 外部路由示例 | 内部路由示例 |
| :--- | :--- | :--- | :--- |
| **`index`** | 分页列表查询 | `/web-api/v1/user/index` | - |
| **`list`** | 全量列表/字典 (不分页) | `/web-api/v1/common/enums` | `/inner-api/v1/user/list` |
| **`detail` / `info`** | 单条详情查询 | `/web-api/v1/user/detail` | `/inner-api/v1/user/info` |
| **`create` / `start`** | 新建资源 / 开启任务 | `/web-api/v1/user/create` | `/inner-api/v1/task/start` |
| **`update`** | 更新已有资源 | `/web-api/v1/user/update` | - |
| **`delete` / `close`** | 删除 / 结束关闭 | `/web-api/v1/user/delete` | `/inner-api/v1/task/close` |
| **`status`** | 状态更新/启停切换 | `/web-api/v1/user/status` | - |

### 1.4 传参与雪花精度红线
- 🚨 **禁在 URL Path 传递业务动态 ID**：GET 查询走 Query（`?user_id=1001`），POST/PUT 变更走 JSON Body；
- 🚨 **64 位长整型精度防护 (`json:",string"`)**：雪花 ID、大整型在 DTO 中必须加 `,string`（如 `json:"id,string"`），防止前端 JS 发生低位截断（$2^{53}-1$ 限制）。普通状态码与分页参数严禁滥加。

---

## 2. API 契约模块化拆分标准 (Modular .api Architecture)

> 🚨 **核心红线**：**严禁单个 `.api` 堆砌全系统契约！必须遵循“入口纯导入 + 按端分目录 + 按领域单一职责”。**

### 2.1 契约工程分层拓扑
```text
app/{service}/
├── api.api                                    🔥 [主入口聚合根] (仅 syntax、info 与 import，通常 < 30 行)
└── desc/                                      📁 [契约领域目录]
    ├── common/common.api                      ├── 公共字典与基础枚举结构
    ├── admin/account.api, report.api          ├── 管理端领域 (按聚合根拆分)
    ├── account/session.api, chat.api          ├── 员工/坐席端领域
    ├── user/chat.api, track.api               ├── 访客/C 端用户领域
    ├── inner/company.api                      ├── 集群内部服务间调用契约
    └── open/open.api                          └── 开放平台 OpenAPI 契约
```

### 2.2 主入口纯导入与子契约规范
- **主入口 `api.api` 纯导入**：严禁声明任何具体的 `type` 或 `service` 块，纯净的 `import "desc/..."`；
- **子契约按聚合根自闭环**：每个 `desc/{端}/{domain}.api` 包含自身 DTO 声明与专属 `@server` 配置；
- **`@server(group)` 严格镜像映射**：`group: admin_account` 必须与生成的 Go 源码目录 1:1 镜像对齐：
  - `internal/handler/admin_account/detail_handler.go`
  - `internal/logic/admin_account/detail_logic.go`
  彻底杜绝根目录下散落几百个 Handler 的混乱局面。

---

## 3. 统一响应与业务异常规范

### 3.1 标准响应体与分页契约
```json
// 基础响应体
{ "code": 0, "msg": "ok", "data": { ... } }

// 分页响应体 (data 节点)
{
  "code": 0, "msg": "ok",
  "data": {
    "page": 1, "page_size": 20, "total": 95, "total_pages": 5,
    "list": [ { "id": "358866610846437376", ... } ]
  }
}
```
> **注**：无数据时 `list` 必须返回空切片 `[]`，严禁返回 `null`。

### 3.2 错误码体系与安全脱敏
1. **统一返回 HTTP 200 + 业务 Code**：严禁向客户端直接裸抛 HTTP 500；
2. **底层 Error 与外发脱敏**：内部通过 `%w` 与 `l.Errorf` 记录完整堆栈；外部一律脱敏，严禁泄露 SQL/Redis/Panic 详情；
3. **分段式错误码矩阵**：
   - `0`：成功；
   - `10001 ~ 10099`：客户端通用错误（参数不合法、未登录、无权限、频控限流）；
   - `20001 ~ 29999`：业务规则冲突（状态机冲突、余额不足、账号冻结）；
   - `30001 ~ 39999`：基础设施异常（统一提示“服务繁忙，请稍后重试”）。

### 3.3 全局统一异常拦截落地
```go
// main.go 注册 go-zero 全局异常处理器
httpx.SetErrorHandlerCtx(func(ctx context.Context, err error) (int, any) {
    if e, ok := err.(*errorx.CodeError); ok {
        return http.StatusOK, types.BaseResponse{Code: e.Code, Msg: e.Msg}
    }
    logx.WithContext(ctx).Errorf("[GlobalError] unhandled: %+v", err)
    return http.StatusOK, types.BaseResponse{Code: 500, Msg: "服务开小差了，请稍后重试"}
})
```

---

## 4. goctl 代码生成与边界控制

1. 🚨 **坚决禁止裸跑散装 `goctl` 命令**：必须通过 `Makefile`（如 `make gen-api` / `make gen-rpc`）标准化执行；
2. **本地模板约束**：统一通过 `--home ./template` 使用本地定制模板，强制生成 `snake_case` 命名风格；
3. **只读保护边界**：`internal/handler/`、`internal/types/`、`*_gen.go` 严禁手工修改，手写业务只允许在 `internal/logic/` 与扩展 `*_model.go` 中编写。

---

## 5. 核心防御性工程铁律

### 5.1 🚨 严禁跨 Logic 相互调用与分层隔离 (Strict Logic Isolation)
- **红线**：严禁在 `Logic A` 中 `new(Logic B)` 或跨包调用同级 Logic；
- **解法**：共享逻辑必须下沉到子领域 Helper（如 `pkg/domain/...`）、Model 扩展方法或注册进 `ServiceContext`。

### 5.2 🚨 API 契约严禁滥用裸 Map (No-Arbitrary-Map)
- **红线**：`.api` 文件的 Request 与 Response 中**坚决禁止声明裸 `map[string]any` 或 `map[string]interface{}`**；
- **要求**：所有接口出入参必须显式定义强类型 Struct，保障契约的单一真理源与客户端 SDK 类型安全。

### 5.3 🚨 API 路由与结构体字段强制全注释 (Mandatory Route & Field Docstrings)
- **核心红线**：`.api` 是前后端团队单一真理源 (SSOT)，**坚决禁止声明任何无注释的裸路由与裸字段**！
- **路由注释规范**：每个接口必须配备 `@doc(summary: "清晰中文业务描述")`，驱动自动生成准确的 Swagger 与 Handler 文档；
- **结构体字段注释规范**：所有 Request 与 Response 的字段，必须逐行标注中文含义、单位、默认值或枚举取值说明（如 `// 客服状态: 1在线, 2忙碌, 3小休, 4离线`），严禁猜谜；
- **标准范式**：
  ```api
  type (
      // CreateSessionReq 访客进线会话创建请求体
      CreateSessionReq {
          SourceGroupId uint64 `json:"source_group_id"` // 进线消息源组全局雪花ID
          SourceType    uint8  `json:"source_type"`     // 渠道类型: 1标准IM, 2第三方设备源
          ClientMeta    string `json:"client_meta"`     // 终端客户端环境元数据(JSON格式: platform/os/browser)
      }
  )

  @server(
      group: user_session
      prefix: /web-api/v1/user
  )
  service chat-api {
      @doc(summary: "访客发起进线建立会话")
      @handler CreateSession
      post /session/create (CreateSessionReq) returns (CreateSessionResp)
  }
  ```

### 5.4 RPC 契约向前兼容
- **Proto 升级规则**：新增字段使用全新 tag 编号，严禁修改已有字段语义或复用已删除编号；废弃字段显式声明 `reserved`。

### 5.5 🚨 公共契约模型严禁命名为 `pkg/types` 反模式 (No pkg/types Anti-Pattern)
- **碰撞机理**：go-zero API 服务的 DTO 由 goctl 生成在 `app/{service}/internal/types`。若全局公共结构体包命名为 `pkg/types`，在 Logic 中同时引用时将触发严重的**包名遮蔽（Shadowing）与别名地狱（如被迫写 `import pkgtypes "..."`）**。
- **强制归拢至 `pkg/schema`**：
  - 接口专属 DTO：收敛于 `internal/types`（只读）；
  - 跨服务复用的通用分页、通用值对象、通用检索：统一收敛至 **`pkg/schema/`**（如 `schema.PageReq`, `schema.PageResult[T]`）；
  - 跨服务 RPC 协议头或异步 MQ 消息载荷：统一收敛至 **`pkg/protocol/`**（如 `protocol.Event`）。
- **标准无冲突范式**：
  ```go
  package logic

  import (
      "context"
      "myproject/app/user/internal/types"  // 本服务专属 HTTP DTO
      "myproject/pkg/schema"              // 全局通用契约模型，天然无别名冲突
  )

  func (l *GetUserListLogic) GetUserList(req *types.GetUserListReq) (*types.GetUserListResp, error) {
      pager := schema.NewPage(req.Page, req.PageSize)
      // ...
      return &types.GetUserListResp{Total: pager.Total}, nil
  }
  ```


