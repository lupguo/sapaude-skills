# Claude Code Skills 管理方法论：从目录到插件

> 将个人 Skills 从 `~/.claude/skills/` 迁移到本地插件的完整方法论，适用于任何需要组织多个 Skill 的场景。

## 为什么要做成插件

### Skills 目录的局限

Claude Code 的 `~/.claude/skills/` 目录有以下限制：

1. **`name` 字段只支持小写字母、数字和连字符** — 冒号（`:`）不合规，三段式命名（如 `brand:category:skill`）会导致发现失败
2. **只自动发现直接子目录** — 嵌套超过一层的 SKILL.md 可能不被框架发现
3. **没有命名空间隔离** — 多个来源的 Skill 混在同一个目录下

### 插件的优势

| 特性 | skills 目录 | 本地插件 |
|---|---|---|
| 命名空间 | 无（靠目录名模拟） | `plugin.json` 的 `name` 字段自动提供 |
| 发现机制 | 只找直接子目录 | skills/ 下标准一层发现 |
| 调用格式 | `/skill-name` | `/plugin-name:skill-name` |
| 分类能力 | 靠嵌套目录（框架不支持） | skill name 前缀（`dev-`、`test-`） |
| 版本管理 | 无 | `plugin.json` 的 `version` 字段 |
| 可移植性 | 绑定 `~/.claude/` | 独立目录，可 `--plugin-dir` 加载 |
| 市场兼容 | 需改造 | 天然兼容插件市场格式 |

## 核心设计

### 1. 命名空间 = plugin.json name

```json
{
  "name": "sapaude",
  "description": "个人技能库",
  "version": "1.0.0"
}
```

这意味着所有 skill 自动获得 `sapaude:` 前缀，调用时写 `/sapaude:skill-name`。

### 2. 分类信息编码在 skill name 中

既然目录只能一层，分类信息通过 **`{lifecycle}-{skill}`** 前缀保留：

```
dev-gorm-entity        # dev = 开发阶段
test-smoke-cycle       # test = 测试阶段
release-chrome-publish # release = 发布阶段
ops-db-migrate         # ops = 运维阶段
```

lifecycle 取值固定 5 个：`plan` / `dev` / `test` / `release` / `ops`。

### 3. 目录结构模式

```
~/.claude/plugins/{brand}/
├── .claude-plugin/
│   └── plugin.json          # 必需：提供命名空间
├── README.md                # 管理规范和技能索引
└── skills/
    ├── {lifecycle}-{name}/  # 每个 skill 一个目录
    │   ├── SKILL.md         # 技能定义（name 字段 = 目录名）
    │   └── ...              # 附属文件（templates/、workflows/ 等）
    └── ...
```

## 迁移步骤

### 从 `~/.claude/skills/` 迁移

假设旧结构是嵌套式：

```
~/.claude/skills/mybrand/
├── category1/
│   └── skill-a/SKILL.md    # name: mybrand:category1:skill-a
├── category2/
│   └── skill-b/SKILL.md    # name: mybrand:category2:skill-b
└── README.md
```

**Step 1: 创建插件骨架**

```bash
mkdir -p ~/.claude/plugins/mybrand/.claude-plugin
mkdir -p ~/.claude/plugins/mybrand/skills
```

**Step 2: 编写 plugin.json**

```json
{
  "name": "mybrand",
  "description": "...",
  "version": "1.0.0",
  "author": { "name": "your-name" }
}
```

**Step 3: 迁移 skill 目录（拍平 + 重命名）**

| 旧路径 | 新路径 | 新 name |
|---|---|---|
| `category1/skill-a/` | `category1-skill-a/` | `category1-skill-a` |
| `category2/skill-b/` | `category2-skill-b/` | `category2-skill-b` |

- 目录名 = SKILL.md 的 name 字段
- 附属文件（templates/、workflows/）跟随迁移

**Step 4: 更新 SKILL.md 的 name 字段**

```yaml
# 旧
name: mybrand:category1:skill-a

# 新
name: category1-skill-a
```

**Step 5: 删除旧目录和 command 文件**

插件体系自动注册 skill 为可调用命令，不再需要 `~/.claude/commands/*.md` 桥接文件。

```bash
rm -rf ~/.claude/skills/mybrand/
rm -f ~/.claude/commands/skill-a.md ~/.claude/commands/skill-b.md
```

**Step 6: 验证**

```bash
claude --plugin-dir ~/.claude/plugins/mybrand
# 在 REPL 中输入 /mybrand:category1-skill-a
```

## 命名规范速查

| 元素 | 格式 | 示例 |
|---|---|---|
| plugin.json name | 全小写，字母数字连字符 | `sapaude` |
| skill name | `{lifecycle}-{skill-name}` | `dev-gorm-entity` |
| skill-name 部分 | kebab-case | `smoke-cycle` |
| 调用命令 | `/plugin:skill-name` | `/sapaude:dev-gorm-entity` |
| 目录名 | 与 skill name 完全一致 | `skills/dev-gorm-entity/` |

## 常见陷阱

### ❌ name 中使用冒号

```yaml
# 错误 — 冒号不是合法字符
name: sapaude:dev:gorm-entity

# 正确 — 只用连字符
name: dev-gorm-entity
```

### ❌ skills/ 下嵌套子目录

```
# 错误 — 框架不发现二级目录
skills/dev/gorm-entity/SKILL.md

# 正确 — 只有一层
skills/dev-gorm-entity/SKILL.md
```

### ❌ 同时维护 command 文件

插件 skill 自动注册为 `/plugin:skill-name` 命令，**不需要**也**不应该**在 `~/.claude/commands/` 下创建桥接文件，否则会造成重复注册和维护负担。

## 扩展：多插件共存

如果你有多个品牌/项目的 skill 集合，每个做成独立插件：

```
~/.claude/plugins/
├── sapaude/          # 个人通用技能
├── team-backend/     # 团队后端规范
└── project-foo/      # 项目特定技能
```

启动时加载多个：

```bash
claude \
  --plugin-dir ~/.claude/plugins/sapaude \
  --plugin-dir ~/.claude/plugins/team-backend
```

或在 `settings.json` 中统一配置。

## 版本历史

| 版本 | 日期 | 说明 |
|---|---|---|
| 1.0 | 2026-03-24 | 初版：从 skills 目录迁移到本地插件 |
