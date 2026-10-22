# 数据库存储、goctl sqlx、GORM 与迁移治理参考手册

## 1. 持久层选型：`goctl + sqlx` vs `GORM / GORM Gen`

go-zero 持久层遵循**“高并发基建”**与**“复杂业务编排”**动静分流架构：

| 维度 | 流派 A：`goctl model + sqlx` (内置官方) | 流派 B：`GORM / GORM Gen` (生态扩展) |
| :--- | :--- | :--- |
| **核心优势** | **极致性能、零反射、原生自带行级缓存一致性** (`sqlc`) | **链式查询、开发效率高、强类型代码生成、富关联加载** |
| **最适场景** | **C 端高并发核心链路**：消息管道、网关接入、状态机流转、高频计数 | **B 端复杂业务**：管理后台动态组合筛选、报表统计、批量导入导出 |
| **架构实践** | **双轨协同制 (Hybrid)**：C 端高并发核心读写走 `sqlx`；复杂后台走 `GORM Gen`，共享 MySQL 连接池。 |

---

## 2. `goctl model + sqlx` 企业级工程标杆

### 2.1 临时目录前缀剥离与重命名规范 (`scripts/gen_model.sh`)
带业务前缀的表（如 `rocky_91chat_sessions`）通过临时目录重命名消除无下划线丑陋文件名，且**仅覆盖 `_gen.go`，严格保护已有 `_model.go`**：
```bash
tmp_dir=$(mktemp -d)
goctl model mysql datasource -url="${DB_DSN}" -table="${full_table}" -dir="${tmp_dir}" -c --home="${TEMPLATE_DIR}"
mv "${tmp_dir}/${raw_name}model_gen.go" "pkg/model/${clean_name}_model_gen.go"
sed -i '' 's/^package .*/package model/' "pkg/model/${clean_name}_model_gen.go"
if [ ! -f "pkg/model/${clean_name}_model.go" ]; then
    mv "${tmp_dir}/${raw_name}model.go" "pkg/model/${clean_name}_model.go"
fi
rm -rf "${tmp_dir}"
```

### 2.2 缓存一致性多 Key 双删与跨 Model 事务 (`InTx`)
```go
// 1. 显式收集多 Key 传入 ExecCtx 执行自动双删
idKey := fmt.Sprintf("%s%v", cacheSessionsIdPrefix, session.Id)
sessionKey := fmt.Sprintf("%s%v", cacheSessionsSessionIdPrefix, sessionId)
_, err = m.ExecCtx(ctx, func(ctx context.Context, conn sqlx.SqlConn) (sql.Result, error) {
    query := fmt.Sprintf("UPDATE %s SET `status` = ?, `updated_at` = ? WHERE `id` = ?", m.table)
    return conn.ExecCtx(ctx, query, status, time.Now(), session.Id)
}, idKey, sessionKey)

// 2. 跨 Model 事务原子绑定 (显式透传 sqlx.Session)
err := l.svcCtx.SessionsModel.TransactCtx(ctx, func(ctx context.Context, session sqlx.Session) error {
    if err := l.svcCtx.SessionsModel.UpdateStatusInTx(ctx, session, sessionId, newStatus); err != nil {
        return err
    }
    return l.svcCtx.LatestProjectionModel.UpsertInTx(ctx, session, projectionData)
})
```

### 2.3 CAS 乱序消息乐观守卫
高并发长连接乱序到达场景，利用 MySQL 原子 `IF` 表达式无锁守卫：
```sql
INSERT INTO user_entry_latest_sessions (company_id, user_id, latest_session_id, last_msg_time)
VALUES (?, ?, ?, ?)
ON DUPLICATE KEY UPDATE
    latest_session_id = IF(VALUES(last_msg_time) >= last_msg_time, VALUES(latest_session_id), latest_session_id),
    last_msg_time = IF(VALUES(last_msg_time) >= last_msg_time, VALUES(last_msg_time), last_msg_time),
    updated_at = NOW();
```

---

## 3. GORM / GORM Gen 黄金守则

1. **防 SQL 注入**：严禁 `Where(fmt.Sprintf(...))`，强制参数化 `Where("user_id = ?", id)`；
2. **防零值丢失**：更新 `0 / false / ""` 必须通过 `Select("field")` 或 `map[string]any`；
3. **独立生成器**：代码生成器入口置于 `cmd/gen/generate.go`，避免污染生产依赖；
4. **标准闭包事务**：`db.WithContext(ctx).Transaction(func(tx *gorm.DB) error { ... })`。

---

## 4. 数据库迁移 (Migration) 与存量洗数红线

### 4.1 `deploy/migrations/` 三层管控架构
- `schema/`：结构变更 DDL（`*.up.sql` / `*.down.sql`），严格纳入 Git 追踪与 CI/CD 自动化应用；
- `data/`：历史数据清洗补丁（`*.sql`）；
- `README.md`：记录数据库变更版本日志。

### 4.2 存量数据清洗四大绝命红线
1. 🚨 **严禁单条大事务更新**：严禁无游标 `UPDATE users SET ...`，必引发从库复制延迟与锁表；
2. 🚨 **主键游标分批 (Chunking)**：每次仅更新 500~1000 条（`WHERE id >= ? AND id < ?`）；
3. 🚨 **批次微休止 (Throttling)**：批次间强制 `sleep 50ms` 让路在线流量；
4. 🚨 **必须可重入 (Idempotency)**：必须带未清洗状态过滤（`WHERE is_migrated = 0`），中断重跑零副作用。

---

## 5. 软删除与唯一键冲突“原子复活 (Revival)”协议

> 🚨 **核心红线**：严禁为了避让软删除而删除物理唯一键 (`UNIQUE KEY`)！

### 5.1 冲突原子复活 SOP
1. **保留物理唯一索引**：表结构强制保留物理唯一键兜底并发防重；
2. **捕获 1062 判定**：新建时捕获 MySQL 1062 冲突：
   - 冲突行 `is_deleted == 0`：说明业务真实重复，抛出业务错误；
   - 冲突行 `is_deleted == 1`：判定为历史已删数据冲突，**触发原子复活**；
3. **复活与责任人流转**：
   - 抹除删除状态：`is_deleted = 0`, `deleted_at = NULL`；
   - 覆写最新属性：更新为当前请求配置；
   - 审计流转：**保留原始 `created_at` 与 `created_by`，当前操作人记为 `updated_by`，`updated_at = NOW()`**。

### 5.2 核心主实体“审计六件套”标准 DDL
```sql
`created_at`  datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
`created_by`  bigint unsigned NOT NULL DEFAULT 0 COMMENT '创建人ID',
`updated_at`  datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
`updated_by`  bigint unsigned NOT NULL DEFAULT 0 COMMENT '更新人ID',
`deleted_at`  datetime DEFAULT NULL COMMENT '软删除时间戳',
`is_deleted`  tinyint unsigned NOT NULL DEFAULT 0 COMMENT '逻辑删除: 0正常, 1已删除',
KEY `idx_is_deleted` (`is_deleted`)
```

---

## 6. 按月动态分表与自愈机制 (Monthly Sharding)

1. **统一工具计算表名**：`GetOpLogTableName(t time.Time)` 格式化物理表名（如 `operation_logs_202610`）；
2. **预建 + 写入自愈**：每月 25 日预建下月表；写入捕获 `Error 1146 (Table doesn't exist)` 异常时自动补建表并重试；
3. **跨月查询约束**：跨月查询采用 `UNION ALL`，业务层强制限制跨度不超过 3 个月。

---

## 7. 领域枚举与状态机规范
关于枚举强类型三层映射（MySQL `tinyint unsigned` $\leftrightarrow$ Go 强类型 $\leftrightarrow$ API `string`）、零值安全两分法则与驱动实现，详见专门手册：
👉 [08_domain_enums_system.md](08_domain_enums_system.md)

---

## 8. 数据库 JSON 字段强类型治理与严禁裸拼接 JSON 规范 (Database JSON Handling)

### 8.1 🚨 绝对红线
1. **严禁手工裸拼接 JSON 字符串**：坚决禁止使用 `fmt.Sprintf("{\"id\":%d,...}", ...)` 拼接 JSON 写入数据库或响应外层，防止逃逸转义 Bug、破坏性非法 JSON 格式与潜在注入风险；
2. **严禁在 Model 中将 JSON 列声明为裸 `string` 或 `map[string]any`**：坚决禁止让业务层反序列化成为随处可见的黑盒逻辑与脆弱断言；所有结构化字段必须声明具体的强类型 Go Struct。

### 8.2 生产级标杆范式：`sql.Scanner` 与 `driver.Valuer` 自动驱动映射
针对 MySQL 8.0 `JSON` 字段定义强类型结构体，将序列化/反序列化彻底内聚于数据层生命周期内：

```go
package model

import (
	"database/sql/driver"
	"encoding/json"
	"fmt"
)

// UserSetting 用户个性化配置实体(映射 user_settings 表中的 json 列 config)
type UserSetting struct {
	Theme         string `json:"theme"`          // 主题样式: light / dark
	NotifyEnabled bool   `json:"notify_enabled"` // 是否开启系统桌面通知
	MaxWindowNum  int    `json:"max_window_num"` // 最大并发会话工作窗口数
}

// Value 实现 driver.Valuer 契约：将结构体自动序列化后存入 MySQL JSON 字段
func (u UserSetting) Value() (driver.Value, error) {
	return json.Marshal(u)
}

// Scan 实现 sql.Scanner 契约：从 MySQL JSON 列读取时自动反序列化到结构体
func (u *UserSetting) Scan(value any) error {
	if value == nil {
		*u = UserSetting{}
		return nil
	}
	bytes, ok := value.([]byte)
	if !ok {
		return fmt.Errorf("failed to scan UserSetting: expected []byte, got %T", value)
	}
	return json.Unmarshal(bytes, u)
}
```

> 💡 **生态集成提示**：
> - 若使用 `GORM`，可直接利用官方泛型封装 `datatypes.JSONType[UserSetting]`（同样实现了底层驱动 Scanner/Valuer），开箱即用且零样板代码；
> - 在 Logic 与 Model 扩展方法中，直接读取 `user.Setting.Theme`，杜绝手动 `json.Unmarshal`。


