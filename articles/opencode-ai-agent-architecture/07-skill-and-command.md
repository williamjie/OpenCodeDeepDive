# 第七篇: 技能系统（Skill System）与命令系统

## 7.1 Skill 系统架构

核心文件：`packages/core/src/skill.ts`

Skill 是可复用的知识/指令单元。每个 Skill 是一个 `SKILL.md` 文件。

### Skill 来源

```typescript
class Source = Union([
  DirectorySource,  // 本地目录搜索 {*.md, **/SKILL.md}
  UrlSource,        // 远程 URL 拉取
  EmbeddedSource,   // 代码中内嵌
])
```

### Skill 数据结构

```typescript
class Info {
  name: string          // 技能名称
  description: string   // 描述
  slash: boolean        // 可作为斜杠命令
  location: AbsolutePath
  content: string       // 完整内容
}
```

### Skill 文件格式

```markdown
---
name: my-skill
description: This skill does...
---
# 技能内容
实际的指令内容...
```

### 发现与加载流程

```text
配置中读取 skills 数组
  → 扫描本地目录或拉取远程 URL
  → 解析 frontmatter → 名称/描述/slash 标志
  → 缓存加载结果
  → Agent 请求时权限过滤
```

### 权限过滤

```typescript
export const available = (skills, agent) =>
  skills.filter(skill =>
    PermissionV2.evaluate("skill", skill.name, agent.permissions).effect !== "deny"
  )
```

### Skill 缓存

```typescript
const cache = new Map<string, Info[]>()
// 每个 source 只加载一次，缓存复用
```

## 7.2 命令系统

命令定义在 `.opencode/command/`，每个文件一个命令：

### 内置命令一览

| 命令 | 功能 | 模型 |
|---|---|---|
| `commit.md` | Git 提交推送 | kimi-k2.5 |
| `changelog.md` | 生成更新日志 | gpt-5.4 |
| `learn.md` | 提取会话经验到 AGENTS.md | 默认 |
| `spellcheck.md` | 拼写检查 | 默认 |
| `translate.md` | 多语言翻译 | claude-opus-4-8 |
| `issues.md` | GitHub Issues 搜索 | claude-haiku-4-5 |
| `rmslop.md` | 移除 AI 代码异味 | 默认 |
| `ai-deps.md` | AI SDK 依赖检查 | 默认 |

### 命令结构

```yaml
---
description: "find issue(s) on github"
model: opencode/claude-haiku-4-5   # 可选：指定模型
# subtask: true                    # 可选：作为子任务
---
搜索命令的具体 prompt 内容...
$ARGUMENTS  # 用户参数注入
```

---
