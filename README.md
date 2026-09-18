# Go-Zero Development Skill (Go 语言与 go-zero 生产级开发规范技能)

[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Go-Zero Version](https://img.shields.io/badge/go--zero-1.x-brightgreen.svg)](https://github.com/zeromicro/go-zero)
[![Target Agents](https://img.shields.io/badge/Agents-Antigravity%20%7C%20Claude%20Code%20%7C%20Cursor%20%7C%20Codex-purple.svg)](#)

本仓库提供了一套面向 **Go 语言 + go-zero 微服务架构** 的企业级 Agent Skill 规范与工程知识库，全面涵盖架构分层、代码生成边界、枚举三层映射、上帝文件拆分、主流标准响应与分页体、异步 Worker 批处理与控制面、长连接网关与 SeqID 发号冷启动自愈、3 步 ACK 闭环、全链路排障与 5 大物理日志分流、防吞异常场景规范、按月分表自愈、Redis Lua 分布式锁及 Docker 优雅停机 (Graceful Shutdown) 交付标准。

本技能已进行全面**通用化与脱敏处理**，完全兼容主流团队协作与开源社区传播。

---

## 🌟 核心规范亮点 (Highlights)

- **严守生成代码只读边界**：深度对齐 `goctl` / `Makefile` 工作流，严格约束契约文件优先，禁止人工篡改 `internal/handler/` 与 `*_gen.go`。
- **拒绝上帝文件 (No God Files)**：单文件建议 200~300 行，超过 500 行强制拆分；一个 API 接口严格对应一个独立 Logic 文件；契约按子域分层拆解。
- **领域枚举三层映射**：数据库 `tinyint unsigned` $\leftrightarrow$ Go 强类型 Enum $\leftrightarrow$ API JSON 英文语义化字符串，消除硬编码与魔法值。
- **主流统一响应与标准分页**：主流外层响应包装（`code/msg/data`）与通用轻量标准分页体（`page/page_size/total/total_pages/list`），支持主流与企业定制标准灵活对齐。
- **禁止 URL Path 传参**：GET 查询一律用 Query，POST/PUT 业务 ID 必须放在 Request Body 中，规范接口语义。
- **高吞吐批量异步刷盘流水线 (Double-Trigger Batch Flusher)**：支持“数量达标”或“时间到期”双触发批量入库，配合优雅停机排空实现零丢消息。
- **Redis ZSet 零 MySQL 慢查巡检**：定时任务通过 Redis ZSet 时间轴索引拉取到期任务，消除周期性大表扫描。
- **SeqID 单调发号器与冷启动自愈**：Redis `INCR` 驱动，遇缓存失效时自动查 DB 最大序号并经由 Lua 脚本原子初始化，防序号倒挂。
- **长连接生命周期与 3 步 ACK**：客户端唯一 MsgID 幂等、Server ACK $	o$ Peer Delivered ACK $	o$ Read ACK 闭环、多端互斥踢下线 (Kick-out) 与心跳保活。
- **全链路 ULID 追踪与 5 大物理日志分流**：`access.log`、`info.log`、`error.log`、`slow.log`、`stat.log` 物理隔离，敏感数据自动正则掩码脱敏。
- **防吞异常五大场景指南**：详尽规定参数转换、主库流转、旁路审计、微服务间调用、网络推流的错误处置准则与正反例代码。
- **按月分表自愈与 Lua 分布式锁**：单调膨胀表按月动态路由 + Error 1146 建表自愈重试；Redis 分布式锁随机 UUID + Lua 脚本原子释放防误删。
- **Docker PID 1 信号透传与优雅关机**：启动脚本必须使用 `exec` 替换进程，确保 Kubernetes 发送的 `SIGTERM` 信号直接被 Go 进程捕获，杜绝被 `SIGKILL` 强杀。

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
└── README.md                         # 详尽的中文项目说明与章节导航
```

---

## 🚀 安装与使用 (Installation & Usage)

### 1. Antigravity IDE / Antigravity CLI
全局自动生效：
```bash
mkdir -p ~/.gemini/config/skills/go-zero-development
cp SKILL.md ~/.gemini/config/skills/go-zero-development/SKILL.md
```

作为项目工作区本地技能生效：
```bash
mkdir -p .agents/skills/go-zero-development
cp SKILL.md .agents/skills/go-zero-development/SKILL.md
```

### 2. 通过 Agent Skills 包管理器安装
```bash
npx skills add Garfield247/go-zero-development
```

### 3. Claude Code / Cursor / Codex
可直接将 `SKILL.md` 内容复制或链接至项目的 `.cursorrules`、`CLAUDE.md` 或 `AGENTS.md` 作为系统上下文指引。

---

## 📖 核心章节导航 (Catalogue)

| 章节 | 核心模块 | 规范重点 |
| :--- | :--- | :--- |
| **01-02** | 通用规则与项目发现 | 调研先行、既有代码优先、HTTP & RPC 调用链推演 |
| **03** | 全局代码组织与防上帝文件 | **No God Files**、500 行拆分阈值、一个用例一个 Logic 文件、命名双轨分层治理 |
| **04** | 领域枚举强类型规范 | DB `tinyint` $\leftrightarrow$ Go Enum $\leftrightarrow$ API JSON 字符串三层映射，禁魔法值 |
| **05** | API 接口与路由设计规范 | `/web-api/` vs `/inner-api/` 隔离、标准动词、**禁止 URL Path 传参**、64 位 ID `,string` |
| **06** | 主流标准响应与分页体 | `code/msg/data` 外层封装、轻量分页体规范、主流 HTTP 映射 / 5 位分段业务错误码体系 |
| **07-09** | Handler / Logic / ServiceContext | 薄 Handler、Logic 职责单一、单例依赖注入、禁全局可变 DB 变量 |
| **10-12** | goctl 与 RPC 通信 | 模板化受控生成、生成物只读铁律、Proto 向前兼容与 `reserved` 规范 |
| **13-16** | 数据库与按月分表自愈 | `utf8mb4_unicode_ci`、GORM 零值与事务、**月度分表自愈重试**、MongoDB 规范 |
| **17** | Redis 缓存与 Lua 分布式锁 | 命名空间多环境隔离 (`local/dev/test/stage/prod`)、TTL 约束、**随机 UUID + Lua 脚本原子释放** |
| **18** | 防吞异常五大场景处置标准 | **Zero Blank Identifier for Error**、五大场景处置标准与典型正反例代码对比 |
| **19-20** | Context 与并发安全红线 | 显式第一参数传递、`pkg/ctxdata` 强类型提取、**新启动协程必须 defer-recover 拦截 Panic** |
| **21** | 异步 Worker 控制面与批处理 | **双触发批量异步刷盘 (Double-Trigger Batch Flusher)**、**Redis ZSet 零慢查巡检**、K8s 探针端点 |
| **22** | 长连接网关与实时通信架构 | **SeqID 发号器冷启动自愈**、**3 步 ACK 闭环**、多端互斥踢下线 (Kick-out)、心跳探测 |
| **23** | 全链路排障与物理日志分流 | **ULID 全链路追踪**、**5 大物理日志分流 (`access/info/error/slow/stat`)**、敏感数据动态脱敏 |
| **24** | Docker 容器化与优雅停机 | 多阶段构建、**PID 1 信号透传 (`exec`)**、优雅关机与缓冲区 Flush |
| **25-29** | 租户隔离、性能、测试与风格 | SQL 强制 `tenant_id`、禁无脑 `SELECT *`、表格驱动测试、代码注释规范 |
| **30-33** | AI 避坑指南、DoD 与黄金准则 | AI 8 大低级错误剖析、13 项完工验收清单、**既有项目代码是第一事实来源** |

---

## ⚖️ 黄金准则 (The Golden Rule)

> **既有项目代码是第一事实来源 (The existing project is the primary source of truth)。**  
> 动手造轮子前，先在已有项目中找示例；  
> 修改生成的代码前，先找到它的定义源头；  
> 引入第三方库前，先检索仓库既有依赖；  
> 更改接口契约前，先评估所有调用方影响；  
> 始终保持改动：**最小精准、语义明确、测试覆盖、稳定兼容！**

---

## 📄 开源许可证 (License)

本项目基于 [MIT License](LICENSE) 开源。
