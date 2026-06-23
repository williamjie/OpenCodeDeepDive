# 第二篇: AI Agent 系统设计与实现

## 2.1 Agent 核心模型

Agent 定义位于两个层面：
- **核心层**: `packages/core/src/agent.ts` — `AgentV2`，定义领域模型和基本操作
- **应用层**: `packages/opencode/src/agent/agent.ts` — 应用层实现，增加运行时配置和 Agent 生成能力

### V2 Core Agent (`packages/core/src/agent.ts`)

```typescript
export const ID = Schema.String.pipe(Schema.brand("AgentV2.ID"))
export const defaultID = ID.make("build")

export class Info extends Schema.Class<Info>("AgentV2.Info")({
  id: ID,                                    // 唯一标识符 (e.g., "build", "plan")
  model: ModelV2.Ref.pipe(Schema.optional),   // 关联模型引用
  request: ProviderV2.Request,                // Provider 请求参数
  system: Schema.String.pipe(Schema.optional),// 系统提示词
  description: Schema.String.pipe(Schema.optional), // 描述
  mode: Schema.Literals(["subagent", "primary", "all"]), // 运行模式
  hidden: Schema.Boolean,                     // 是否对用户隐藏
  color: Color.pipe(Schema.optional),         // UI 显示颜色
  steps: PositiveInt.pipe(Schema.optional),   // 最大迭代步数
  permissions: PermissionSchema.Ruleset,      // 工具权限规则集
})
```

### Agent 模式说明

| 模式 | 含义 | 可见性 | 用途 |
|---|---|---|---|
| **primary** | 主 Agent | UI 可见，可切换 | `build`、`plan` |
| **subagent** | 子 Agent | 内部使用 | `explore`、`general` |
| **all** | 两者兼可 | 视具体情况 | 用户自定义 Agent |

### Core Agent 服务接口

```typescript
interface Interface {
  get: (id: ID) => Effect.Effect<Info | undefined>         // 按 ID 获取
  default: () => Effect.Effect<Info | undefined>           // 获取默认 Agent
  resolve: (id?: ID) => Effect.Effect<Info | undefined>    // 解析（ID 或默认）
  select: (id?: ID) => Effect.Effect<Selection>            // 选择 Agent
  all: () => Effect.Effect<Info[]>                          // 列出所有
}
```

### 应用层 Agent (`packages/opencode/src/agent/agent.ts`)

应用层扩展了更多运行时字段：

```typescript
export const Info = Schema.Struct({
  name: Schema.String,
  description: Schema.optional(Schema.String),
  mode: Schema.Literals(["subagent", "primary", "all"]),
  native: Schema.optional(Schema.Boolean),  // 是否使用原生运行时
  hidden: Schema.optional(Schema.Boolean),
  topP: Schema.optional(Schema.Finite),
  temperature: Schema.optional(Schema.Finite),
  color: Schema.optional(Schema.String),
  permission: PermissionV1.Ruleset,         // V1 兼容权限规则
  model: Schema.optional(Schema.Struct({
    modelID: ModelV2.ID,
    providerID: ProviderV2.ID,
  })),
  variant: Schema.optional(Schema.String),  // 模型变体
  prompt: Schema.optional(Schema.String),    // 系统提示词
  options: Schema.Record(Schema.String, Schema.Unknown), // Provider 选项
  steps: Schema.optional(Schema.Finite),
})
```

## 2.2 内置 Agent 详解

OpenCode 内置了 7 个 Agent，定义在 `packages/opencode/src/agent/agent.ts` 的 `agents` 对象中：

### build — 默认开发 Agent

```typescript
build: {
  name: "build",
  description: "The default agent. Executes tools based on configured permissions.",
  permission: Permission.merge(defaults, {
    question: "allow",     // 允许询问用户
    plan_enter: "allow",   // 允许进入计划模式
  }, user),
  mode: "primary",
  native: true,
}
```

### plan — 计划/只读 Agent

```typescript
plan: {
  name: "plan",
  description: "Plan mode. Disallows all edit tools.",
  permission: Permission.merge(defaults, {
    question: "allow",
    plan_exit: "allow",
    task: { general: "deny" },
    edit: { "*": "deny" },  // 禁止所有编辑工具
    external_directory: { [data/plans/*]: "allow" }
  }, user),
  mode: "primary",
  native: true,
}
```

### general — 通用子 Agent

```typescript
general: {
  name: "general",
  description: "General-purpose agent for researching complex questions and executing multi-step tasks.",
  permission: Permission.merge(defaults, { todowrite: "deny" }, user),
  mode: "subagent", native: true,
}
```

### explore — 代码探索专用 Subagent

```typescript
explore: {
  name: "explore",
  permission: Permission.merge(defaults, {
    "*": "deny", grep: "allow", glob: "allow", list: "allow",
    bash: "allow", webfetch: "allow", websearch: "allow", read: "allow",
    external_directory: readonlyExternalDirectory,
  }, user),
  prompt: PROMPT_EXPLORE,  // 自定义系统提示词
  mode: "subagent", native: true,
}
```

### compaction / title / summary — 隐藏 Agent

```typescript
compaction: { mode: "primary", hidden: true, prompt: PROMPT_COMPACTION, permission: { "*": "deny" } }
title: { mode: "primary", hidden: true, temperature: 0.5, permission: { "*": "deny" }, prompt: PROMPT_TITLE }
summary: { mode: "primary", hidden: true, permission: { "*": "deny" }, prompt: PROMPT_SUMMARY }
```

### 内置 Agent 对比表

| Agent | 模式 | 隐藏 | 权限策略 | 系统提示词 | 用途场景 |
|---|---|---|---|---|---|
| **build** | primary | ❌ | 全放开 + 用户确认 | 无 | 日常编码开发 |
| **plan** | primary | ❌ | 禁止编辑工具 | 无 | 代码审查与规划 |
| **general** | subagent | ❌ | 禁止 todowrite | 无 | 多步骤任务 |
| **explore** | subagent | ❌ | 只读（read/grep/glob） | PROMPT_EXPLORE | 代码搜索 |
| **compaction** | primary | ✅ | 禁止所有 | PROMPT_COMPACTION | 上下文压缩 |
| **title** | primary | ✅ | 禁止所有 | PROMPT_TITLE | 标题生成 |
| **summary** | primary | ✅ | 禁止所有 | PROMPT_SUMMARY | 会话总结 |

## 2.3 用户自定义 Agent 配置

通过 `opencode.jsonc` 的 `agents` 字段自定义：

```jsonc
{
  "agents": {
    "build": { "model": "anthropic/claude-sonnet-4-20250514" },
    "reviewer": {
      "model": "openrouter/openai/gpt-5",
      "description": "Review changes",
      "system": "You are a code reviewer...",
      "mode": "subagent",
      "color": "warning",
      "steps": 12,
      "disabled": false,
      "permissions": [{ "action": "edit", "resource": "*", "effect": "deny" }]
    }
  }
}
```

### 配置覆盖规则

```
1. disable: true → 删除该 Agent
2. 同名存在 → 合并覆盖（model, prompt, description 等替换，permission 合并）
3. 同名不存在 → 创建新的自定义 Agent
4. 确保 Truncate.GLOB 始终允许
```

## 2.4 Agent 生命周期

```text
opencode.jsonc → Config.Service → AgentV2.layer → LocationServiceMap → 运行时可查询
```

Agent 选择逻辑：
```typescript
select(id?) → 1) 指定 ID 查找  2) 配置默认  3) build  4) 第一个可见 primary
```

---
