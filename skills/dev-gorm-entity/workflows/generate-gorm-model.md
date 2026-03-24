# Command: Generate GORM Model

## 📌 用途

为新的业务对象创建符合 GORM 规范的完整 Go struct。

## 🚀 使用方法

在 Claude Code 中输入：
```
/generate-gorm-model
```

然后描述你想要建模的业务对象。

## 💡 使用示例

### 示例 1：简单的用户模型
```
/generate-gorm-model

我需要一个 Category（分类）模型，包含以下字段：
- 分类名称 (name) - 必填，最多 100 字符，要求唯一
- 描述 (description) - 可选的长文本
- 是否启用 (is_enabled) - 默认为启用
- 排序权重 (sort_order) - 整数，默认 0
- 可以按名称查询，也经常按启用状态过滤
```

### 示例 2：包含复杂类型的模型
```
/generate-gorm-model

Product（产品）模型：
- 名称 (name) - 必填，唯一
- SKU - 必填，唯一
- 价格 (price) - decimal
- 库存 (inventory) - 整数
- 分类 ID (category_id) - 外键索引
- 标签 (tags) - 字符串数组，JSON 存储
- 属性 (attributes) - 复杂对象，JSON 存储
```

### 示例 3：复合查询的模型
```
/generate-gorm-model

Order（订单）模型：
- 订单号 (order_no) - 必填，唯一
- 用户 ID (user_id) - 必填
- 状态 (status) - pending/processing/shipped/delivered，需要索引
- 总金额 (total_amount) - decimal
- 订单项 (items) - JSON 数组（包含 product_id, quantity, price）
- 配送地址 (shipping_address) - JSON 对象

常见查询：
- 按用户 ID 和状态查询订单
- 按订单号精确查询
```

## 📋 生成内容

我将为你生成：

✅ **完整的 Go struct**
- 遵循 GORM 规范的 5 个核心点
- 完整的字段标签（column, type, not null, default, comment）
- 正确的字段顺序（ID → 业务字段 → 时间戳）

✅ **Tabler 接口实现**
```go
func (CategoryName) TableName() string {
    return "t_category_names"
}
```

✅ **索引配置**
- 唯一约束使用 `uniqueIndex:uq_*`
- 查询索引使用 `index:idx_*`
- 复合索引使用 `priority` 指定顺序

✅ **JSON 序列化**
- 数组字段自动使用 `datatypes.JSONSlice`
- 对象字段自动使用 `datatypes.JSONType`
- 正确的 `serializer:json` 标签

✅ **详细注释**
- 每个字段都有中文 comment 说明
- 解释设计决策

## ⚡ 快速参考

| 字段类型 | 数据库类型 | GORM 标签示例 |
|---------|----------|------------|
| 字符串（短） | varchar(255) | `type:varchar(255)` |
| 字符串（ID） | varchar(50) | `type:varchar(50)` |
| 整数 | int | `type:int` |
| 布尔 | tinyint(1) | `type:tinyint(1)` |
| 小数 | decimal(10,2) | `type:decimal(10,2)` |
| 长文本 | text | `type:text` |
| 数组 | json | `serializer:json` |
| 对象 | json | `serializer:json` |

## 📝 提示

- **尽可能详细** - 描述越详细，生成的模型越精准
- **指出查询模式** - 告诉我常见的 WHERE 条件，我会配置适当的索引
- **提及约束** - 如果有字段必须唯一或互斥，明确说出来
- **考虑扩展** - 如果字段将来可能增长，告诉我（便于选择字符串长度）

## 🔍 检查生成的模型

收到代码后，对照检查清单验证：

- [ ] ID 是第一个字段，使用 `primaryKey` 标签
- [ ] 业务字段在中间，时间戳在最后
- [ ] 所有字段都有完整的 gorm 标签
- [ ] 表名使用 `t_` 前缀
- [ ] 实现了 Tabler 接口
- [ ] 唯一字段使用 `uniqueIndex:uq_*`
- [ ] 常查询的字段使用 `index:idx_*`
- [ ] 复杂类型使用 `serializer:json`

## 🎯 下一步

生成后，你可以：

1. **直接使用** - 将代码复制到你的项目
2. **修改调整** - 如需调整，告诉我具体改动
3. **验证规范** - 使用 `/validate-gorm-model` 命令验证
4. **迁移现有代码** - 使用 `/migrate-to-gorm-spec` 命令升级旧模型

## 📚 更多帮助

- `gorm-entity-design` skill - 查看完整的规范文档
- `database-design-principles` - 理解设计哲学
- `index-strategy` - 深入了解索引策略