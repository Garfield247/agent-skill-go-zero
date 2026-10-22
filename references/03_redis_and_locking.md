# Redis 高并发缓存与原子分布式锁参考手册

## 1. 命名空间与多环境隔离规范 (Environment-Aware Key Namespace)

为了兼容**多套环境共用同一 Redis 实例**（如本地联调与测试共用、灰度演练与正式共用）的隔离需求，Redis Key 的命名空间采用分层分段规范：

### 1.1 标准命名格式
```text
{project}:[{env}:]{module}:{object}:{identifiers...}
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
3. **独立集群可选省略**：在生产环境拥有物理完全隔离的独立 Redis 集群时，环境标签可选省略（保持 `{project}:{module}:{object}:{identifier}`）。

---

### 1.2 🚨 严禁裸拼 Key：统一工具方法集中生产规范 (Key Generation Protocol)

> 🚨 **核心红线**：**坚决禁止在业务 Logic、Handler 或 Model 中到处手写裸字符串拼接 Key（如 `fmt.Sprintf("rocky:user:%d", id)` ❌）！**
> 
> **违背红线的次生危害**：
> 1. 分隔符（`:`）与字段顺序混乱，极易引发键冲突；
> 2. 无法自动附着多环境隔离前缀，极易发生本地/测试数据击穿或污染生产；
> 3. 散落各处的硬编码使代码无法通过 IDE / LSP 符号引用快速反查 Key 的全部读写链路。

#### 生产级工程标杆范式 (`pkg/util/redisx/key.go`)

所有业务 Key 必须集中收敛于公共工具包，由强类型具名函数统一生成：

```go
package redisx

var globalPrefix = "rocky:91chat" // main.go 启动时通过 c.Redis.KeyPrefix 动态注入

// BuildKey 底层通用构造器: 自动附着多环境前缀 {prefix}:{module}:{object}:{ids...}
func BuildKey(module, object string, ids ...any) string {
	parts := append([]string{globalPrefix, module, object}, toStr(ids)...)
	return strings.Join(parts, ":")
}

// ---------------- 业务层唯一合法入口：具名工具方法 (严禁手写裸拼) ----------------
func KeyPresence(role string, companyID, targetID any) string { return BuildKey("presence", role, companyID, targetID) }
func KeyAccountStatus(companyID, accountID any) string        { return BuildKey("account", "status", companyID, accountID) }
func KeyLock(business string, uniqueID any) string           { return BuildKey("lock", business, uniqueID) }
```

---

## 2. TTL（生命周期）约束
- **严禁生成无 TTL 的垃圾缓存**：除少数全局发号序列器外，所有业务缓存必须显式设置过期时间（TTL）；
- 在线长连接或会话路由 Key 兜底设置 TTL（由心跳周期性 `EXPIRE` 续期）。

---

## 3. 安全分布式锁标准实现 (UUID + Lua)
- **Lock Key 格式**：强制通过 `redisx.KeyLock(business, uniqueID)` 生成；
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

