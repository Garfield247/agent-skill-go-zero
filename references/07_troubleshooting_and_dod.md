# 性能调优、排障武器库与交付基准 (DoD) 参考手册

## 1. 生产排障与调试武器库

在排查 Go 与 go-zero 微服务生产缺陷时，必须运用以下核心调试武器库：

### 1.1 并发竞态与协程泄漏排查
- **动态竞态检测**：本地或 CI 中针对并发改动强制启用 `-race`：
  ```bash
  go test -race -v ./...
  ```
- **Goroutine 堆栈转储与泄漏诊断**：内存缓慢爬升或连接不释放时，引入 `net/http/pprof` 检查：
  ```bash
  curl -s http://127.0.0.1:6060/debug/pprof/goroutine?debug=2 | grep -A 10 "goroutine"
  ```
  排查重点：检查是否有无缓冲 Channel 缺乏接收方导致生产者永久阻塞，或 `context.WithCancel` 遗漏 `defer cancel()`。

### 1.2 慢调用与性能瓶颈诊断 (Slow Queries & Bottlenecks)
- **go-zero 框架慢日志**：框架默认对 >500ms 的 API/RPC 请求及 >100ms 的 DB 操作输出 `[SLOW-SQL]` 或 `[SLOW-RPC]` 告警日志，直接通过上下文 TraceID 提取全链路日志；
- **连接池饱和排查**：若出现 `context deadline exceeded` 或 `bad connection`，检查 `MaxOpenConns` / `MaxIdleConns` 配置，并利用 `EXPLAIN` 检查慢查行数与索引命中。

### 1.3 缓存击穿与分布式锁排查
- **缓存穿透/击穿**：高频失效后防 DB 穿透，必须使用 SingleFlight（go-zero 内置 `collection.NewCache`）；
- **锁自动释放排查**：持有锁的业务逻辑若执行耗时超过锁 TTL 会发生静默释放，必须保障看门狗续期或严格的超时时间上限。

---

## 2. 性能调优与多租户安全准则

1. **防深度分页 (Keyset Pagination)**：数百万级大表严禁大 Offset，强制采用游标推进：
   ```sql
   WHERE id > ? ORDER BY id ASC LIMIT ?
   ```
2. **严禁无脑 `SELECT *`**：高频链路必须明确指定投影列（如 `Select("id", "status")`）；
3. **多租户隔离防越权**：
   - 租户 ID 必须从 Context 中安全提取，严禁直接信任前端传参；
   - 业务 CRUD SQL 必须显式携带 `company_id = ?`（或 `tenant_id = ?`），严防跨租户越权。

---

## 3. 交付完成定义 (Definition of Done - DoD)

任务交付验收前，对照以下清单逐项自检：

- [ ] **契约先行**：接口变动已正确更新 `.api` / `.proto`，并通过 Makefile 重新生成，未手改生成文件；
- [ ] **分层纯净**：业务逻辑收敛于 Logic 层，Handler 仅做参数校验与响应包装，未跨 Logic 相互调用；
- [ ] **单一职责与防上帝文件**：单文件尽量控制在 500 行内（超 1000 行酌情拆分，硬顶超 1500 行必须拆分，生成代码与测试豁免），一个 API 对应一个 Logic 文件；
- [ ] **健壮防腐**：全链路雪花 ID 添加 `json:",string"`，可选字段使用指针，未滥用裸 Map；
- [ ] **错误防吞**：除测试用例外，严禁使用 `_` 忽略有价值的 error，跨层 `%w` 包装，业务层禁主动 `panic()`；
- [ ] **并发安全**：后台协程首行包含 `defer-recover` 兜底，并发改动通过 `go test -race ./...` 检验；
- [ ] **缓存与事务**：手写更新方法显式收集多 Key 双删，多表联动使用 `*InTx` 事务原子绑定；
- [ ] **代码整洁**：执行 `gofmt -w .`，未引入全库大面积格式化噪音，无明文密钥泄露。
