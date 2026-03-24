---
name: dev-new-docs
description: 使用项目规约构建新功能、漏洞修复或重构分支
argument-hint: [ feature|bugfix|refactor|simple ] [description] [ empty|template ]
allowed-tools: Bash, Write, Read
model: haiku
---

# 创建指定类型的文档

- TaskType: **$1**
- TaskDescription: **$2**
- DocContent: **$3**

`{DocContent}` 默认为 `empty`。

- TaskTitle: 基于用户输入的{TaskDescription}内容进行标题生成

## 执行步骤

### empty 模式 (默认) - 1 次工具调用

单个 Bash 命令完成所有操作：

```bash
mkdir -p docs/{TaskType} && echo "# {TaskType} {TaskTitle}" > "docs/{TaskType}/{TaskType}_{TaskTitle}_$(date +%y%m%d%H%M%S).md"
```

### template 模式 - 2 轮工具调用

**Step 1**: 并行执行

1. Bash: `mkdir -p docs/{TaskType} && date +%y%m%d%H%M%S`
2. Read: `templates/{TaskType}.md`

**Step 2**: Write 文件

- 路径: `docs/{TaskType}/{TaskType}_{TaskTitle}_{时间戳}.md`

## 模板文件

- `templates/feature.md` - 新特性需求
- `templates/bugfix.md` - 问题修复需求
- `templates/refactor.md` - 重构需求
- `templates/simple.md` - 简单需求

## 示例

### empty 模式 (1 次调用)

```
> /new_docs feature 用户登录
```

执行: `mkdir -p docs/feature && echo "# feature 用户登录" > "docs/feature/feature_用户登录_$(date +%y%m%d%H%M%S).md"`

### template 模式 (2 轮调用)

```
> /new_docs feature 用户登录 template
```

执行:

1. 并行: Bash + Read
2. Write
