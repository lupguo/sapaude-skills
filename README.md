# Sapaude Skills

> 个人 Claude Code 技能集合 Marketplace，覆盖 plan/dev/test/release/ops 全生命周期。

## 安装

```bash
# 添加 marketplace
/plugin marketplace add lupguo/sapaude-skills

# 安装插件
/plugin install sapaude@sapaude-skills
```

## 插件列表

### sapaude

个人技能库，包含以下技能：

| 命令 | 说明 |
|---|---|
| `/sapaude:dev-gorm-entity` | GORM 实体设计规范（5 核心规则） |
| `/sapaude:dev-new-docs` | 基于模板创建项目文档（feature/bugfix/refactor/simple） |
| `/sapaude:test-smoke-cycle` | 通用冒烟测试循环（build→start→test→stop→clean） |
| `/sapaude:release-chrome-publish` | Chrome 扩展发布流水线（esbuild→CHANGELOG→ZIP→CWS） |

## 添加新插件

以 hugo-blog 为例：

```bash
# 1. 创建插件目录
mkdir -p plugins/hugo-blog/.claude-plugin
mkdir -p plugins/hugo-blog/skills/dev-new-post

# 2. 创建 plugin.json
echo '{"name":"hugo-blog","description":"Hugo 博客工作流","version":"1.0.0"}' \
  > plugins/hugo-blog/.claude-plugin/plugin.json

# 3. 在 .claude-plugin/marketplace.json 的 plugins 数组中添加：
# {"name":"hugo-blog","source":"hugo-blog","description":"Hugo 博客工作流"}

# 4. 安装
/plugin install hugo-blog@sapaude-skills
```

## 目录结构

```
sapaude-skills/
├── .claude-plugin/
│   └── marketplace.json        # marketplace 目录（列出所有插件）
├── plugins/
│   ├── sapaude/                # 插件 1
│   │   ├── .claude-plugin/plugin.json
│   │   └── skills/...
│   └── hugo-blog/              # 插件 2（示例）
│       ├── .claude-plugin/plugin.json
│       └── skills/...
├── README.md
├── LICENSE
└── .gitignore
```

## 生命周期分类

| 前缀 | 阶段 | 说明 |
|---|---|---|
| `plan-` | 想 | 技术调研、架构选型 |
| `dev-` | 写 | 遵循模式或生成代码 |
| `test-` | 验 | 验证代码是否正确 |
| `release-` | 发 | 交付发布 |
| `ops-` | 维 | 上线后维护 |

## License

MIT
