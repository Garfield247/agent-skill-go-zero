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

## 📦 安装与多 Agent 使用指南 (Installation & Multi-Agent Usage)

本项目遵循开放 Agent 规范，支持在 **Gemini / Antigravity**、**Anthropic Claude**、**Cursor / Codex** 等各类主流 Agent 环境中一键安装与激活：

### 1. Google Antigravity / Gemini Code Assist
- **全局安装（推荐）**：
  ```bash
  git clone git@github.com:Garfield247/agent-skill-go-zero.git ~/.gemini/config/skills/go-zero-development
  ```
- **项目工作区局部引入**：
  ```bash
  mkdir -p .agents/skills
  git clone git@github.com:Garfield247/agent-skill-go-zero.git .agents/skills/go-zero-development
  ```

### 2. Anthropic Claude (Claude Code / Claude Projects)
- **Claude Code (CLI 终端智能体)**：
  克隆至 Claude 全局技能库：
  ```bash
  mkdir -p ~/.claude/skills
  git clone git@github.com:Garfield247/agent-skill-go-zero.git ~/.claude/skills/go-zero-development
  ```
  *或者在项目根目录的 `CLAUDE.md` 中追加引入：*
  ```markdown
  See detailed engineering specifications in: ~/.claude/skills/go-zero-development/SKILL.md
  ```
- **Claude Projects (Web / 桌面端)**：
  直接将仓库中的 `SKILL.md` 内容复制并粘贴至 Project 的 **Project Knowledge (项目知识库)** 或 **Custom Instructions (自定义指令)** 中。

### 3. Cursor / GitHub Copilot / OpenAI Codex
- **Cursor (现代 MDC 规则体系)**：
  在项目根目录创建或链接规则：
  ```bash
  mkdir -p .cursor/rules
  # 克隆或软链接为 Cursor 专有规则文件
  git clone git@github.com:Garfield247/agent-skill-go-zero.git .cursor/rules/go-zero-development
  ```
- **GitHub Copilot / Codex**：
  将本技能规范注入 Copilot 指令集：
  ```bash
  mkdir -p .github
  cat << 'EOF' >> .github/copilot-instructions.md
  # 引入本技能核心规则
  EOF
  cat path/to/SKILL.md >> .github/copilot-instructions.md
  ```

## 📄 开源协议 (License)
本项目采用 [MIT License](LICENSE) 授权。
