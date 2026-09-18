# agent-skill-go-zero

> ⚡ Go 语言与 go-zero 高性能微服务架构工程规范与 Agent Skill。涵盖 Handler/Logic/Model 四层分层职责、统一标准响应与错误码体系、并发安全与 Panic 拦截、Redis 缓存与分环境命名空间等全套生产级最佳实践。

## 🌟 仓库与规范说明 (Repository Description)

本项目是专为 **AI Agent（如 Google Antigravity、Gemini Code Assist、Claude、Cursor）** 以及研发工程师结对打造的高性能微服务开发技能库。通过系统化、结构化的工程规则，规范代码生成边界，防范生产级隐患。

### 核心规范模块
1. **统一契约与工具链**：严格遵循 `goctl` 代码生成契约，严禁手动篡改 `*_gen.go` 文件。
2. **严格四层分层**：
   - `Handler`：仅处理入参解析、校验与统一输出包装，严禁编写业务逻辑。
   - `Logic`：纯业务编排，日志全链路携带 `ctx context.Context`。
   - `ServiceContext`：资源持有层（DB、Redis、RPC 客户端依赖注入）。
   - `Model`：数据持久层，扩展方法分离至单独扩展文件。
3. **统一响应与错误码体系**：
   - 标准统一响应体：`{"code": 0, "msg": "ok", "data": ...}`。
   - 错误码遵循主流五位分段语义（`1xxxx` 通用错误、`2xxxx` 用户认证、`3xxxx` 业务参数校验、`5xxxx` 外部与系统故障）。
4. **并发安全与 Panic 拦截红线**：所有新启动的 Goroutine 必须显式捕获 `recover()`，严禁未捕获 panic 击垮主进程。
5. **多环境隔离与缓存治理**：Redis Key 采用主流标准化多环境分段命名空间格式（`{project}:{env}:{module}:{business_key}`），彻底杜绝本地与测试环境串用。

## 📦 安装与加载 (Installation)

### 方式 1: 安装至 Antigravity / Gemini 全局技能库
```bash
git clone git@github.com:Garfield247/agent-skill-go-zero.git ~/.gemini/config/skills/go-zero-development
```

### 方式 2: 在任意微服务项目中作为本地工作区技能引入
```bash
mkdir -p .agents/skills
git clone git@github.com:Garfield247/agent-skill-go-zero.git .agents/skills/go-zero-development
```

## 📄 开源协议 (License)
本项目采用 [MIT License](LICENSE) 授权。
