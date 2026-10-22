# 错误处理、并发安全与 Context 萃取参考手册

## 1. 错误处理与防吞异常工程红线 (Zero Blank Identifier)

> 🚨 **核心红线**：生产代码中**一律严禁使用空标识符 `_` 静默忽略 `error` 或具有业务语义的返回值**！

### 1.1 五大场景标准化处置准则
1. **数据转换与解析 (`time.Parse`, `json.Unmarshal`)**：严禁忽略解析错误，必须阻断并返回参数异常，防非法零值流入业务深层；
2. **核心业务流转 (DB / Redis 更新)**：严禁降级静默忽略，失败必须阻断主流程并触发事务回滚；
3. **旁路与异步任务 (审计日志、事件广播)**：允许主流程继续，但必须使用 `l.Errorf("[模块] 描述: err=%v", err)` 显式留痕；
4. **微服务间调用 (Inner-API / RPC)**：严禁丢弃 SDK 返回的 error，即使在协程中也必须记录带业务 ID 的错误日志；
5. **网络推流与 I/O (WebSocket)**：推流失败记录 `l.Errorf`；推流成功但对端未在线（`count == 0`）记录 `l.Debugf` 标记离线。

### 1.2 错误包装与禁止主动 Panic
- **错误包装**：跨层向上冒泡错误必须使用 `fmt.Errorf("...: %w", err)` 携带上下文；
- **严禁主动 Panic**：业务层与 Logic 层中**坚决禁止主动调用 `panic()`**，一律返回业务错误码。

---

## 2. Context 上下文传递与强类型萃取规范

1. **显式第一参数**：涉及 I/O、RPC、数据库操作的方法，首参数必须为 `ctx context.Context`；
2. **严禁断流**：链路中严禁用 `context.Background()` 替换上游传递的 `r.Context()` 或 `l.ctx`，防止 TraceID 与超时机制丢失；
3. **强类型萃取器 (`pkg/ctxdata`)**：严禁手写 `ctx.Value(key).(string)` 裸断言（防 Panic 崩溃），统一封装类型安全 Getter：
   ```go
   package ctxdata
   func GetUserID(ctx context.Context) (int64, bool) {
       val, ok := ctx.Value(CtxKeyUserID).(int64)
       return val, ok
   }
   ```
4. **网关头注入与防伪**：网关鉴权后注入私有头（如 `X-User-ID`），内部微服务拦截器必须校验网关签名，严防外网伪造。

---

## 3. 并发控制与 Goroutine 安全红线

1. 🚨 **Goroutine Panic 拦截铁律**：所有新启动的后台 Goroutine，**首行必须通过 `defer-recover` 拦截 Panic**：
   ```go
   go func() {
       defer func() {
           if r := recover(); r != nil {
               logx.WithContext(ctx).Errorf("goroutine panic recovered: %v, stack: %s", r, debug.Stack())
           }
       }()
       // 异步逻辑...
   }()
   ```
2. **循环变量闭包防坑**：并发协程消费循环项时，显式作为函数参数传入（Go 1.22 以下版本防脏写）：
   ```go
   for _, item := range list {
       go func(val ItemType) { process(val) }(item)
   }
   ```
3. **竞态检测门禁**：改动涉及并发逻辑时，本地必须运行 `go test -race ./...` 检验通过。
