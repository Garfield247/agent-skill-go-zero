# Go-Zero Development Skill (Go 语言与 go-zero 微服务开发技能)

[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Go-Zero Version](https://img.shields.io/badge/go--zero-1.x-brightgreen.svg)](https://github.com/zeromicro/go-zero)
[![Target Agents](https://img.shields.io/badge/Agents-Antigravity%20%7C%20Claude%20Code%20%7C%20Cursor%20%7C%20Codex-purple.svg)](#)

本仓库提供了一套专门面向 **Go 语言 + go-zero 微服务架构** 的高质量 Agent Skill 规范与工程知识库，全面涵盖了基于 go-zero 框架进行高性能后端开发时的架构边界、代码生成约束、数据库事务、缓存、并发安全与测试标准。

---

## 🌟 核心特性 (Features)

- **43 大核心规范模块**：从通用规则、项目结构推演、`.api` / `.proto` 契约定义，到 Handler / Logic / ServiceContext / Model 分层落地。
- **严守代码生成边界**：深度对齐 `goctl` 自动化工作流，严格约束“源头定义优先、生成代码只读”，彻底解决 AI 乱改 generated code 的通病。
- **全方位基础设施覆盖**：涵盖 GORM / GORM Gen（防注入与零值更新）、MongoDB、Redis 缓存、事务与最终一致性、多租户数据隔离。
- **并发与安全红线**：强制规定 Goroutine Panic 拦截（`defer-recover`）、循环变量安全、显式 Context 传递与结构化并发。
- **AI 常见误区与避坑指南 (Mistakes to Avoid)**：详细列举了 AI 编写 go-zero 时的 8 大高频低级错误，并给出明确避坑方案。
- **闭环行动流 (DoD & Agent Workflow)**：提供完备的 Definition of Done 交付清单与 7 步标准化开发行动闭环。

---

## 📂 仓库结构 (Repository Layout)

```text
.
├── SKILL.md                          # 根目录 Agent Skill 核心规范说明文件 (支持单 Skill 载入)
├── skills/
│   └── go-zero-development/
│       └── SKILL.md                  # 符合多 Skill 规范管理目录结构的技能文件
├── .gitignore                        # Git 忽略规则
├── LICENSE                           # MIT 开源许可证
└── README.md                         # 项目中文说明文档
```

---

## 🚀 安装与使用 (Installation & Usage)

### 1. Antigravity IDE / Antigravity CLI
将本技能克隆或软链接到全局配置目录即可全局自动生效：

```bash
# 全局生效路径
mkdir -p ~/.gemini/config/skills/go-zero-development
cp SKILL.md ~/.gemini/config/skills/go-zero-development/SKILL.md
```

或者作为项目工作区技能（Workspace Skill）：
```bash
mkdir -p .agents/skills/go-zero-development
cp SKILL.md .agents/skills/go-zero-development/SKILL.md
```

### 2. 通过 Agent Skills 包管理器安装
如果使用标准 Agent Skills 工具链：
```bash
npx skills add Garfield247/-go-zero-development
```

### 3. Claude Code / Cursor / Codex
可直接将 `SKILL.md` 内容复制或引入至项目的 `.cursorrules`、`CLAUDE.md` 或 `AGENTS.md` 中作为系统上下文规则。

---

## 📖 核心章节导航 (Catalogue)

| 章节编号 | 规范名称 | 核心要点 |
| :--- | :--- | :--- |
| **01-02** | 通用规则与项目发现 | 修改前调研、既有代码优先、HTTP & RPC 调用链全景推演 |
| **03-05** | API / Handler / Logic | API 为单一事实来源、`,string` 精度保护、Thin Handler、Logic 纯粹性 |
| **06-08** | ServiceContext 与 goctl | 统一单例依赖注入、goctl 生成受控、生成代码只读边界 |
| **09** | RPC / zrpc 服务通信 | 契约向前兼容、禁止直连跨服务 DB、Protobuf `reserved` 规范 |
| **10-13** | 存储层与缓存 | MySQL / GORM 零值与事务、MongoDB Context、Redis Key与TTL |
| **14-17** | 错误、Context 与并发 | `%w` 错误包装、禁止 `_` 忽略、Goroutine Panic 拦截与防闭包捕获 |
| **18-23** | 兼容性、鉴权与租户 | 接口只增不改、从 Context 安全提取身份、多租户强制隔离 |
| **24-28** | 性能、查询与测试 | 避免无脑 `SELECT *`、Keyset 深度分页、表格驱动测试与 `-race` 检查 |
| **29-35** | 重构、依赖、Docker 与 CI | 小步重构、多阶段 Dockerfile、GitLab CI 标准流程、扁平化代码风格 |
| **36-40** | 避坑指南与 DoD | AI 8 大避坑高频误区、代码检索策略、Git 安全、17 条交付自检清单 |
| **41-43** | Agent 行为流与黄金准则 | 7 步标准化闭环、决策优先级矩阵、**既有项目代码是第一事实来源** |

---

## ⚖️ 黄金准则 (The Golden Rule)

> **既有项目代码是第一事实来源 (The existing project is the primary source of truth)。**  
> 动手造轮子前，先在已有项目中找示例；  
> 修改生成的代码前，先找到它的定义源头；  
> 引入第三方库前，先检索仓库既有依赖；  
> 始终保持改动：**最小精准、语义明确、测试覆盖、稳定兼容！**

---

## 📄 开源许可证 (License)

本项目基于 [MIT License](LICENSE) 开源。
