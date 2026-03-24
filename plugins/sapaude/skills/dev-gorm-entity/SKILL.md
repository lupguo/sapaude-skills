---
name: dev-gorm-entity
description: |
  GORM entity model design following gorm.io/gorm best practices for Go.
  Use when creating Go struct models for databases, defining database entities, 
  reviewing GORM data structures for specification compliance, or migrating existing models.
  Enforces 5 core rules: table naming (t_ prefix), field ordering, tag specifications, 
  indexing strategy (idx_/uq_ prefix), and JSON serialization for complex types.
---

# GORM 数据库实体模型设计规范

## 概述

本 Skill 定义了使用 `gorm.io/gorm` 库创建 Go 结构体时的企业级标准。遵循这些规范确保团队代码的一致性、可维护性和性能。

---

## 核心规范 - 5 个关键点

### 1. 表与结构体对应 - 实现 Tabler 接口

**原则**：每个 Go 结构体应唯一对应一个数据库表。

**实现方式**：

- 通过实现 `gorm.Tabler` 接口
- 实现 `TableName() string` 方法来明确表名
- **表名统一使用 `t_` 前缀** 格式（例如 `t_users`, `t_orders`）

**代码示例**：

```go
type User struct {
    ID    uint
    Email string
    // ... 其他字段
}

// 实现 Tabler 接口
func (User) TableName() string {
    return "t_users"
}
```

**为什么使用 t\_ 前缀？**

- 清晰区分业务表和系统表
- 提高 SQL 查询可读性
- 避免与 SQL 关键字冲突
- 便于表名搜索和过滤

---

### 2. 标准结构体字段顺序

**原则**：字段排列顺序必须严格遵循以下顺序，确保团队内一致性。

**标准顺序**：

```
ID (主键)
  ↓
业务字段1
业务字段2
...
业务字段N
  ↓
CreatedAt (创建时间)
UpdatedAt (更新时间)
DeletedAt (软删除标记)
```

**代码示例**：

```go
type User struct {
    // 主键
    ID uint `gorm:"primaryKey"`

    // 业务字段
    Email    string
    Username string
    Password string
    IsActive bool
    Role     string

    // 时间戳字段（自动管理）
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt
}
```

**为什么这样排序？**

- **ID 优先** - 主键必须首先出现，便于快速识别
- **业务字段居中** - 核心数据集中在中间
- **时间戳末尾** - 系统字段放在最后，不混淆业务逻辑
- **可预测性** - 团队成员期望这样的顺序，提升代码可读性

---

### 3. 完备的 GORM 字段标签

**原则**：所有业务字段必须配置完整的 `gorm` 标签。

**必需标签属性**：

| 属性       | 说明                                         | 示例                        |
| ---------- | -------------------------------------------- | --------------------------- |
| `column`   | 数据库列名（通常与字段名一致，但用蛇形命名） | `column:user_email`         |
| `type`     | 数据库字段类型                               | `type:varchar(255)`         |
| `not null` | 非空约束                                     | `not null`                  |
| `default`  | 默认值                                       | `default:''` 或 `default:0` |
| `comment`  | 字段描述（提高维护性）                       | `comment:用户邮箱`          |

**数据类型约定**：

- **字符串（短）** - 最大长度 < 255 字符时：`varchar(255)` ⚠️ 注意：即使不需要 255 字符，也统一使用 varchar(255) 方便未来扩展
- **字符串（长）** - 最大长度 > 255 字符时：`text` 或 `longtext`
- **整数** - `int` 或 `bigint`
- **小数** - `decimal(10,2)` 或 `float`
- **布尔值** - **统一使用 `tinyint(1)`**（MySQL 不原生支持 boolean）
- **日期时间** - `timestamp` 或 `datetime(3)` 用于毫秒精度
- **JSON** - `json` 或 `jsonb`（PostgreSQL）

**完整标签示例**：

```go
type User struct {
    ID uint `gorm:"primaryKey"`
    Email string `gorm:"column:email;type:varchar(255);not null;default:'';comment:用户邮箱"`
    Phone string `gorm:"column:phone;type:varchar(20);default:'';comment:用户手机号"`
    IsActive bool `gorm:"column:is_active;type:tinyint(1);not null;default:1;comment:是否激活"`
    Age int `gorm:"column:age;type:int;default:0;comment:用户年龄"`
    Bio string `gorm:"column:bio;type:text;comment:个人简介"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt
}
```

**标签最佳实践**：

1. 始终指定 `column` - 不要依赖 GORM 的自动蛇形转换
2. 始终指定 `type` - 显式声明数据库类型
3. 为业务关键字段指定 `not null` - 数据完整性
4. 为大多数字段指定 `default` - 防止 NULL 侵入
5. 为每个字段添加 `comment` - 这是代码注释，也是数据字典

---

### 4. 业务驱动的数据库索引

**原则**：根据实际业务查询需求和数据约束配置索引。

**索引类型**：

#### 普通索引（`index`）- 用于查询优化

用于经常被查询的字段：

- WHERE 子句中的条件字段
- ORDER BY 中的排序字段
- JOIN 条件中的字段
- GROUP BY 中的字段

**命名规范**：`idx_字段名` 或 `idx_组合字段名`

```go
type User struct {
    ID       uint
    Email    string `gorm:"index:idx_email"`        // 按邮箱查询用户
    Phone    string `gorm:"index:idx_phone"`        // 按手机查询用户
    IsActive bool   `gorm:"index:idx_is_active"`   // 查询活跃用户

    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt
}
```

#### 唯一索引（`uniqueIndex`）- 用于数据约束

用于必须唯一的字段，防止重复数据：

- 用户邮箱（一个邮箱只能注册一个账户）
- 用户手机号（一个手机号只能注册一个账户）
- API Key（每个 Key 唯一）
- 订单号（每个订单号唯一）

**命名规范**：`uq_字段名` 或 `uq_组合字段名`

```go
type User struct {
    ID       uint
    Email    string `gorm:"uniqueIndex:uq_email;type:varchar(255);not null"`
    Phone    string `gorm:"uniqueIndex:uq_phone;type:varchar(20)"`
    Username string `gorm:"uniqueIndex:uq_username;type:varchar(100);not null"`

    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt
}
```

#### 复合索引 - 多字段组合查询

当业务需要在多个字段组合上查询时：

```go
type Order struct {
    ID      uint
    UserID  uint `gorm:"index:idx_user_created,priority:1"`
    Status  string `gorm:"index:idx_user_created,priority:2"`

    // 业务查询：WHERE user_id = ? AND status = 'pending'
    // 一个复合索引同时优化这个查询

    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt
}
```

**索引决策流程**：

1. **识别查询模式** - 哪些 WHERE 条件最常用？
2. **考虑写性能** - 过多索引会减慢插入/更新
3. **评估数据量** - 小表可能不需要索引
4. **定期审查** - 业务变化后，索引策略也要调整

---

### 5. 复杂数据类型序列化 (JSON)

**原则**：对于数组或自定义值对象，使用 JSON 序列化存储在数据库的单个文本字段中。

**适用场景**：

- 存储数组：`[]string`, `[]int`, `[]ProductID`
- 存储对象：`map[string]interface{}`, 自定义 struct
- 存储关联数据：避免多表 JOIN 的场景

**实现方式**：
使用 `gorm:"serializer:json"` 标签

**类型支持**：

```go
import "gorm.io/datatypes"

type Product struct {
    ID       uint

    // 字符串数组
    Tags     datatypes.JSONSlice `gorm:"column:tags;type:json;serializer:json;comment:产品标签"`

    // 自定义 struct 数组
    Images   datatypes.JSONType `gorm:"column:images;type:json;serializer:json;comment:图片列表"`

    // 自定义 struct 对象
    Metadata datatypes.JSONType `gorm:"column:metadata;type:json;serializer:json;comment:元数据"`

    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt
}
```

**代码示例**：

```go
// 定义自定义类型
type Image struct {
    URL     string `json:"url"`
    AltText string `json:"alt_text"`
    Width   int    `json:"width"`
    Height  int    `json:"height"`
}

type Product struct {
    ID    uint
    Name  string

    // JSON 序列化示例
    Tags   datatypes.JSONSlice `gorm:"column:tags;type:json;serializer:json;comment:产品标签"`
    Images datatypes.JSONType  `gorm:"column:images;type:json;serializer:json;comment:产品图片"`

    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt
}

func (Product) TableName() string {
    return "t_products"
}

// 使用示例
product := Product{
    Name: "Laptop",
    Tags: []string{"electronics", "computers"},  // 自动序列化为 JSON
    // Images 可以存储复杂结构
}
db.Create(&product)  // 自动 JSON 序列化
```

**注意事项**：

1. ✅ 使用 `gorm.io/datatypes` 中的类型便于 GORM 自动处理
2. ✅ JSON 字段在查询时性能一般，不建议频繁在 WHERE 子句中用 JSON 路径查询
3. ✅ 对于频繁查询的字段，仍应该规范化到单独的列
4. ✅ JSON 字段适合存储"元数据"、"配置"、"附加信息"

---

## 快速检查清单

创建或审查 GORM 模型时，使用此清单确保完全符合规范：

```go
// 示例模型
type Article struct {
    // [ ] ID 是第一个字段，使用 uint 类型和 primaryKey 标签
    ID uint `gorm:"primaryKey"`

    // [ ] 所有业务字段都有完整的标签
    Title   string `gorm:"column:title;type:varchar(255);not null;comment:文章标题"`
    Content string `gorm:"column:content;type:text;not null;comment:文章内容"`

    // [ ] 根据业务需求配置了索引
    // [ ] 唯一约束使用 uniqueIndex 和 uq_ 前缀
    // [ ] 普通索引使用 index 和 idx_ 前缀
    AuthorID uint   `gorm:"column:author_id;index:idx_author_id;comment:作者 ID"`
    Tags     string `gorm:"column:tags;type:json;serializer:json;comment:文章标签"`

    // [ ] 时间戳字段在最后，顺序为 CreatedAt → UpdatedAt → DeletedAt
    CreatedAt time.Time      `gorm:"column:created_at;autoCreateTime:milli"`
    UpdatedAt time.Time      `gorm:"column:updated_at;autoUpdateTime:milli"`
    DeletedAt gorm.DeletedAt `gorm:"column:deleted_at;comment:软删除"`
}

// [ ] 实现 Tabler 接口，TableName() 返回 t_ 前缀的表名
func (Article) TableName() string {
    return "t_articles"
}
```

**验证步骤**：

1. ✅ 实现了 `Tabler` 接口，`TableName()` 返回 `t_` 前缀表名
2. ✅ 字段顺序：ID → 业务字段 → CreatedAt → UpdatedAt → DeletedAt
3. ✅ 所有业务字段有完整 GORM 标签（column, type, not null, default, comment）
4. ✅ 布尔值使用 `tinyint(1)`，短字符串使用 `varchar(255)` 或 `varchar(20)` 等
5. ✅ 根据业务需求配置了索引（`idx_` 或 `uq_` 前缀）
6. ✅ 数组/复杂类型使用了 `serializer:json`
7. ✅ 每个字段都有有意义的 `comment`

---

## 常见问题

### Q: 为什么一定要用 t\_ 前缀？

**A**:

- 清晰标识业务表 vs 系统表
- 避免与 SQL 关键字冲突（如 `user`, `order` 等）
- 便于数据库管理员快速识别应用表
- 团队统一规范，减少争议

### Q: 如果字段可能为空，是否应该使用 `not null`？

**A**:

- ✅ **强烈建议**：尽量不允许 NULL
- 使用 `default` 值代替 NULL（空字符串、0、false 等）
- 这样更便于业务逻辑，避免处理 NULL 值的复杂性
- 特殊情况（如软删除 DeletedAt）可以允许 NULL

### Q: varchar(255) 是固定长度吗？

**A**:

- ❌ 不是。MySQL 中 varchar(255) 表示**最多** 255 字符
- 实际占用空间取决于数据长度（加 1-2 字节开销）
- varchar(20) 对 20 字符短字符串更经济
- 但为简化，我们统一使用 varchar(255) 作为通用字符串类型

### Q: 一定要为每个字段添加 default 吗？

**A**:

- ❌ 不一定，但**强烈建议**
- 对于业务字段：提供合理的默认值
- 对于必填字段：如果没有默认值，应在应用层确保值的完整性
- 默认值在数据库层提供保护，是防御性编程

### Q: 何时使用唯一索引 vs 唯一约束？

**A**:

- GORM 中 `uniqueIndex` 同时创建索引和唯一约束
- 这是最佳实践：既优化查询，又约束数据
- 不需要分别指定 `constraint` 和 `index`

### Q: 复合索引的 priority 是什么？

**A**:

- `priority` 指定索引中字段的顺序
- MySQL 复合索引遵循**最左前缀规则**
- 示例：`(user_id, status)` 可优化：
  - `WHERE user_id = ?`
  - `WHERE user_id = ? AND status = ?`
  - 但**不能**优化：`WHERE status = ?` 单独查询

### Q: 能否在 JSON 字段上创建索引？

**A**:

- ❌ 传统索引不适用
- 现代数据库（MySQL 5.7+, PostgreSQL）支持 **JSON 路径索引**
- 如果需要频繁按 JSON 内容查询，建议将该字段规范化为独立列
- JSON 最适合存储非结构化或不常查询的数据

---

## 迁移与版本

### 当前版本：v1.0

**适用 GORM 版本**：v1.25+

### 何时更新规范

- GORM 重大版本发布时（v1 → v2 → v3）
- 新的数据库功能出现时（如 JSON Schema Validation）
- 团队实践发生重要变更时

---

## 相关资源

- **GORM 官方文档**：https://gorm.io/docs
- **GORM 字段标签文档**：https://gorm.io/docs/models.html
- **MySQL 数据类型**：https://dev.mysql.com/doc/refman/8.0/en/data-types.html
- **JSON 序列化最佳实践**：https://gorm.io/docs/serializer.html

---

## 扩展阅读

详细的话题讨论，请参考 context 目录中的文档：

- `database-design-principles.md` - 数据库设计哲学
- `index-strategy.md` - 深入索引策略
- `migration-guide.md` - 从现有代码迁移到规范

## 示例代码

完整的真实示例代码，请参考 examples 目录：

- `user-model.go` - 用户模型（基础示例）
- `order-model.go` - 订单模型（复合索引 + JSON 序列化）
- `advanced-patterns.go` - 高级模式和最佳实践

---

## 获得帮助

当你需要：

- **生成新模型** - 告诉我你的业务对象，我将生成符合规范的 Go struct
- **验证现有模型** - 粘贴你的代码，我将检查符合性并提供改进建议
- **迁移旧代码** - 提供现有模型，我将逐步重构到符合规范

只需告诉我你的需求，这个 Skill 会自动应用！
