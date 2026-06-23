# 第七篇: 技能系统（Skill System）与命令系统

> **用人话说**：Skill 是 Agent 的"技能包"，Command 是 Agent 的"快捷指令"。

---

## Skill 系统

### Skill 是什么？

Skill 是可复用的知识/指令单元，每个 Skill 是一个 `SKILL.md` 文件。

用人话说：就像一个"操作手册"，告诉 Agent 遇到特定情况该怎么做。

### Skill 来源

| 来源 | 用人话说 |
|---|---|
| DirectorySource | 本地目录搜索 |
| UrlSource | 远程 URL 拉取 |
| EmbeddedSource | 代码中内嵌 |

### Skill 数据结构

| 字段 | 用人话说 |
|---|---|
| name | 技能名称 |
| description | 描述 |
| slash | 可作为斜杠命令 |
| location | 文件位置 |
| content | 完整内容 |

### Skill 文件格式

```
---
name: my-skill
description: This skill does...
---
# 技能内容
实际的指令内容...
```

**用人话说**：就像一个 Markdown 文件，头部是元数据，正文是具体指令。

### 发现与加载流程

```
配置中读取 skills 数组
  → 扫描本地目录或拉取远程 URL
  → 解析 frontmatter → 名称/描述/slash 标志
  → 缓存加载结果
  → Agent 请求时权限过滤
```

**用人话说**：就像应用商店——先扫描有哪些技能，然后按权限过滤，最后缓存起来。

### 权限过滤

不是所有 Agent 都能用所有 Skill。系统会根据 Agent 的权限规则过滤：

```
Skill 列表 → 按权限过滤 → 可用 Skill 列表
```

**用人话说**：就像员工权限——不是所有人都能用所有工具。

### Skill 缓存

每个 Skill 只加载一次，缓存复用。

**用人话说**：就像浏览器缓存，第一次加载慢，后面就快了。

---

## 命令系统

### Command 是什么？

Command 定义在 `.opencode/command/` 目录，每个文件一个命令。

用人话说：就像快捷指令，输入 `/commit` 就能执行 Git 提交。

### 内置命令一览

| 命令 | 功能 | 用人话说 |
|---|---|---|
| `commit.md` | Git 提交推送 | 提交代码 |
| `changelog.md` | 生成更新日志 | 写更新记录 |
| `learn.md` | 提取会话经验到 AGENTS.md | 总结经验 |
| `spellcheck.md` | 拼写检查 | 检查错别字 |
| `translate.md` | 多语言翻译 | 翻译文档 |
| `issues.md` | GitHub Issues 搜索 | 查 issue |
| `rmslop.md` | 移除 AI 代码异味 | 清理 AI 生成的代码 |
| `ai-deps.md` | AI SDK 依赖检查 | 检查依赖 |

### 命令结构

```
---
description: "find issue(s) on github"
model: opencode/claude-haiku-4-5   # 可选：指定模型
---
搜索命令的具体 prompt 内容...
$ARGUMENTS  # 用户参数注入
```

**用人话说**：命令就是"预设的 prompt"，输入命令就自动执行对应的 prompt。

---

## Skill vs Command

| 维度 | Skill | Command |
|---|---|---|
| 触发方式 | Agent 自动加载 | 用户手动输入 `/命令` |
| 用途 | 知识/指令 | 快捷操作 |
| 文件位置 | 任意目录 | `.opencode/command/` |
| 参数 | 无 | 支持 `$ARGUMENTS` |

**用人话说**：Skill 是"教科书"，Agent 自己读；Command 是"快捷键"，用户自己按。

---

## 一句话总结

Skill 是 Agent 的"技能包"，通过文件发现和加载，支持权限过滤；Command 是"快捷指令"，用户输入命令就自动执行预设的 prompt。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-ai-agent-architecture/07-skill-and-command.md)*
