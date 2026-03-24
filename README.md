# Sapaude Skills

> 个人技能库插件，覆盖 plan/dev/test/release/ops 全生命周期。

## 安装

```bash
# 添加 marketplace
/plugin marketplace add lupguo/sapaude-skills

# 安装插件
/plugin install sapaude@sapaude-skills
```

## 可用技能

| 命令 | 说明 |
|---|---|
| `/sapaude:dev-gorm-entity` | GORM 实体设计规范（5 核心规则） |
| `/sapaude:dev-new-docs` | 基于模板创建项目文档（feature/bugfix/refactor/simple） |
| `/sapaude:test-smoke-cycle` | 通用冒烟测试循环（build→start→test→stop→clean） |
| `/sapaude:release-chrome-publish` | Chrome 扩展发布流水线（esbuild→CHANGELOG→ZIP→CWS） |

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
