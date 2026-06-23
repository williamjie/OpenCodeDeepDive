# OpenCode AI Agent 架构深度解析 — 系列文章

> 生成时间: 2026-06-23 08:59
> 文件名: 20260623-0859-opencode-ai-agent-analysis.md
> 生成模型: deepseek-v4-flash-free

---

## 📋 概览

| 项目 | 信息 |
|---|---|
| **项目名称** | OpenCode |
| **开发者** | anomalyco |
| **项目定位** | 开源 AI 编程助手 (The open source AI coding agent) |
| **技术栈** | TypeScript, Bun, Effect (v4), Drizzle ORM, SQLite, AI SDK |
| **包架构** | Monorepo (25 packages)，Turborepo + Bun workspaces |
| **开源协议** | MIT |
| **GitHub** | https://github.com/anomalyco/opencode |

本系列文章通过对 OpenCode 源码的深度解析，揭示一个生产级 AI Agent 系统的完整架构设计，涵盖智能体搭建、子智能体创建、多智能体协作、工具系统、会话管理、技能/插件系统等核心主题。

---

## 📑 文章索引

| 编号 | 文章标题 | 核心内容 |
|---|---|---|
| 一 | OpenCode 架构总览与核心技术栈 | 项目全景、包结构、技术选型、设计哲学 |
| 二 | AI Agent 系统设计与实现 | Agent 核心模型、内置 Agent、自定义配置、生命周期 |
| 三 | Subagent 创建机制深度剖析 | Subagent 概念、内置 Subagent、权限继承、创建流程 |
| 四 | 多智能体协作机制 | Orchestrator-Worker 模式、并行/串行委派、会话协调器 |
| 五 | 工具系统架构 | V2 工具模型、内置工具、注册执行流程 |
| 六 | 会话系统设计 | 会话架构、V2 会话 API、Context Epoch、数据流 |
| 七 | 技能系统与命令系统 | Skill 发现加载、命令定义、权限过滤 |
| 八 | 插件系统扩展开发 | V2 Effect Plugin 架构、插件钩子、热加载 |
| 九 | LLM 运行时与 Provider 集成 | 双层运行时、AI SDK、原生 LLM、Provider 管理 |
| 十 | 技术亮点与最佳实践 | 架构亮点、关注场景、二次开发指南 |

---

# 第一篇: OpenCode 架构总览与核心技术栈

## 1.1 项目全景

OpenCode 是一个完整的开源 AI 编程助手，对标 Claude Code，提供多平台接入能力：

- **TUI 终端界面** (`packages/tui`) — 命令行交互界面，类似 Claude Code 的终端体验
- **Desktop 桌面应用** (`packages/desktop`) — Electron 封装的原生桌面应用
- **Web 控制台** (`packages/console/app`) — 浏览器端控制台
- **Web App** (`packages/web`, `packages/app`) — SolidStart 构建的 Web 应用
- **CLI 命令行** (`packages/cli`) — 命令行入口
- **VSCode SDK 集成** (`sdks/vscode`) — VSCode 扩展
- **JavaScript SDK** (`packages/sdk`, `packages/sdk/js`) — 编程 SDK

## 1.2 Monorepo 包结构

OpenCode 采用 Turborepo + Bun workspaces 管理 25 个包：

### 核心层 (Core)

| 包路径 | 职责 | 依赖等级 |
|---|---|---|
| `packages/core` | **领域模型、Schema、事件系统、会话管理(V2)、工具系统、插件钩子、配置系统** | 核心依赖 |
| `packages/opencode` | **应用层编排、Agent 管理、Provider 适配、会话处理、CLI/Server 入口** | 核心依赖 |
| `packages/llm` | 原生 LLM 运行时 (`@opencode-ai/llm`)，LLMClient / RequestExecutor | 核心依赖 |

### 插件与扩展

| 包路径 | 职责 |
|---|---|
| `packages/plugin` | 插件 SDK，供外部开发者开发插件 |
| `packages/sdk` | SDK 和 API 客户端生成 |
| `packages/script` | 脚本工具 |

### UI 层

| 包路径 | 职责 |
|---|---|
| `packages/tui` | 终端 UI 界面 (SolidJS + opentui) |
| `packages/ui` | 共享 UI 组件库 |
| `packages/web` | Web 应用 (lander/登录页) |
| `packages/app` | Web 主应用 (SolidStart) |
| `packages/console/app` | Console 管理端 |
| `packages/storybook` | 组件文档 (Storybook) |

### 桌面端

| 包路径 | 职责 |
|---|---|
| `packages/desktop` | 桌面应用 (Electron) |

### 基础设施

| 包路径 | 职责 |
|---|---|
| `packages/effect-drizzle-sqlite` | Effect + Drizzle ORM + SQLite 集成 |
| `packages/effect-sqlite-node` | Effect SQLite Node 适配 |
| `packages/identity` | 身份认证服务 |
| `packages/server` | 后端服务器 |
| `packages/stats` | 统计面板 (SST) |
| `packages/http-recorder` | HTTP 录制调试工具 |
| `packages/function` | 云函数 |
| `packages/enterprise` | 企业版功能 |

### 文档与社区

| 包路径 | 职责 |
|---|---|
| `packages/docs` | 项目文档 |
| `packages/containers` | Docker 容器化 |
| `packages/slack` | Slack 集成 |

## 1.3 核心技术栈详解

### TypeScript + Bun

- 运行时: **Bun v1.3.14**（替代 Node.js）
- 语言: 全 TypeScript，开启 strict 模式
- 构建: Turborepo 管理 pipeline
- 测试: `bun test`
- 类型检查: `bun typecheck`（而非 tsc）
- Linter: `oxlint`（替代 ESLint）
- 格式化: Prettier

### Effect v4 函数式编程框架

这是 OpenCode 最核心、最具特色的技术选型。Effect v4 提供了远超 async/await 的能力：

**核心概念映射：**

| Effect 概念 | 类似的传统概念 | 在 OpenCode 中的应用 |
|---|---|---|
| `Effect.gen(function* () { ... })` | async/await + try/catch | 所有异步逻辑编排 |
| `Context.Service` | 依赖注入容器 | 服务定义与获取 |
| `Layer.effect()` | 依赖注册 | 服务实现注册 |
| `Schema.Class()` | Zod/Zod 对象 | 类型定义 + 运行时验证 |
| `Scope` | try/resource | 资源生命周期管理 |
| `SynchronizedRef` | Mutex | 并发状态管理 |
| `Schema.TaggedErrorClass` | 类型错误 | 类型化错误处理 |

**典型服务模式：**
```typescript
// 1. 定义接口
export interface Interface {
  readonly get: (id: ID) => Effect.Effect<Info | undefined>
  readonly list: () => Effect.Effect<Info[]>
}

// 2. 注册 Service
export class Service extends Context.Service<Service, Interface>()("@opencode/v2/Agent") {}

// 3. 实现 Layer
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const config = yield* Config.Service  // 自动注入
    // 实现逻辑...
    return Service.of({ get, list })
  }),
)

// 4. 使用
const agent = yield* AgentV2.Service
const info = yield* agent.get("build")
```

### AI SDK

- **默认运行时**: Vercel AI SDK (`ai` npm package) 的 `streamText` / `generateObject`
- **原生运行时**: `@opencode-ai/llm`（opt-in，通过 `OPENCODE_EXPERIMENTAL_NATIVE_LLM=true` 启用）
- **支持的 Provider**: OpenAI、Anthropic、Google、xAI、OpenRouter 等
- **结构化输出**: 使用 AI SDK 的 `generateObject` 配合 Effect Schema

### Drizzle ORM + SQLite

- **ORM**: Drizzle ORM (v1.0.0-rc.2)
- **数据库**: SQLite (通过 Effect 管理事务)
- **Schema**: 定义在 `packages/core/src/session/sql.ts`

## 1.4 V2 架构设计哲学

从 `specs/v2/instructions.md` 提炼的设计原则：

### 原则一：Core 薄层

> "Move behavior out of large application services and into plugins. Core services should become small, typed containers that own state, expose simple operations, and trigger hooks where policy or integration-specific logic belongs."

Core 包只应包含：
- 领域 Schema 和类型错误
- 状态容器
- 事件定义
- 插件钩子合约

### 原则二：插件化

> "Plugins implement provider-specific, config-specific, auth-specific, model-discovery, and generation behavior."

业务逻辑通过插件实现，核心不包含应用策略。

### 原则三：可重放变换

> "Services are hot-reloadable by design: updates are granular, observable, and do not require tearing down the whole process."

配置变更和插件加载通过可重放的变换（replayable transform）实现。

### 原则四：事件溯源

会话使用事件溯源模式，所有状态变更记录为持久化事件流。

### 原则五：Location 作用域

服务实例按工作目录（Location）缓存不同目录有独立的服务实例和状态。

## 1.5 配置发现机制

OpenCode 支持多级配置，按优先级从低到高：

1. **全局配置**: `~/.config/opencode/opencode.jsonc`
2. **项目配置文件**: 从项目目录往上查找 `opencode.json` / `opencode.jsonc`
3. **`.opencode` 目录**: `{project}/.opencode/opencode.jsonc`（最高优先级）

**规则**：越具体的配置优先级越高，权限规则使用相反的顺序（全局规则可以覆盖项目规则）。

---


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

# 第三篇: Subagent 创建机制深度剖析

## 3.1 Subagent 概念

Subagent 是在主会话上下文中通过 `task` 工具派生的独立 AI 会话：

- **独立会话**: 每个 subagent 创建一个独立会话，拥有自己的上下文窗口
- **独立权限**: 权限受父 Agent 限制 + 自身配置共同决定
- **并行执行**: `run_in_background=true` 支持并行执行
- **会话延续**: `task_id="ses_xxx"` 保留上下文

## 3.2 Subagent vs Primary Agent

| 维度 | Primary Agent | Subagent |
|---|---|---|
| 可见性 | UI 可见，用户可切换 | 内部使用 |
| 调用方式 | 默认 Tab 切换 | `task()` 工具 |
| 生命周期 | 持久会话 | 任务级 |
| 权限继承 | 完整自身权限 | 父 deny + 自身权限 |
| 执行方式 | 串行 | 可串行/并行 |

## 3.3 内置 Subagent

**explore**: 只读搜索专用。仅允许 grep/glob/read/bash/webfetch/websearch，禁止任何写入。
**general**: 通用多步骤任务。全放开（除 todowrite）。

## 3.4 权限继承机制

核心文件：`packages/opencode/src/agent/subagent-permissions.ts`

```typescript
function deriveSubagentSessionPermission(input) {
  // 1. 继承父会话的 deny 规则和 external_directory 规则
  // 2. 子 Agent 没有显式配置 task → 禁止
  // 3. 子 Agent 没有显式配置 todowrite → 禁止
}
```

**核心原则**：父会话的 **deny 规则继承**，allow 规则**不继承**（Subagent 用自己的权限）。

## 3.5 Subagent 创建流程

```text
主会话 → task() 工具
  → 解析参数 (agent/category/prompt/background/task_id)
  → 解析 Agent 配置
  → deriveSubagentSessionPermission()
  → 创建/复用会话
  → 提交 prompt 并执行
  → 后台模式: 返回 bg_xxx，完成后 system-reminder 通知
  → 前台模式: 阻塞等待结果
  → 结果返回主会话
```

## 3.6 Category 到 Agent 映射

| category | 映射 Agent | 用途 |
|---|---|---|
| visual-engineering | 前端专用 | UI/UX/设计 |
| deep | general | 深度研究 |
| quick | 简化版 | 简单修改 |
| explore | explore | 代码搜索 |
| oracle | 只读咨询 | 架构/调试 |
| librarian | 文献搜索 | 外部库查询 |

---

# 第四篇: 多智能体协作机制

## 4.1 Orchestrator-Worker 模式

```text
用户 → 主会话 (build/plan) → Subagent(explore) / Subagent(general) / Tool Calls
```

## 4.2 四种协作模式

**模式一：并行探索** — 同时发起多个 explore 搜索
```typescript
task(subagent_type="explore", run_in_background=true, prompt="...")
task(subagent_type="explore", run_in_background=true, prompt="...")
// 主 Agent 继续工作，完成时收到通知
```

**模式二：串行委派** — 分步骤执行
```typescript
const result = task(agent="general", prompt="第一步研究")
task(task_id=result.ses_id, prompt="第二步实现")
```

**模式三：会话延续** — 保留上下文继续对话
```typescript
const result = task(category="deep", prompt="分析")
task(task_id=result.ses_id, prompt="继续分析...")
```

**模式四：工具共享** — Agent 通过工具系统协作

## 4.3 SessionRunCoordinator

```typescript
interface Interface {
  resume(sessionID): void   // 启动或加入
  wake(sessionID): void     // 唤醒（合并多次）
  interrupt(sessionID): void // 中断
}
// 同一 Session 串行，不同 Session 并行
```

## 4.4 Eager Tool Execution

V2 运行器特性：解析到 tool_call 立即执行，不等 LLM 流结束。

```text
LLM 流 → 解析 tool_call → 立即执行
       → 继续解析流
LLM 流结束 → 等待所有工具 → 持久化 → 判断继续
```

## 4.5 BackgroundJob 系统

```text
start() → running → extend()→ completed/error/cancelled
```

| 方法 | 功能 |
|---|---|
| start/ extend | 启动/延续任务 |
| wait/cancel | 等待/取消 |
| promote/waitForPromotion | 提升到前台 |
| list/get | 列表/查询 |

- 非持久化（内存管理）
- 每个任务有独立 Effect Scope
- 链式延续支持

---


# 第五篇: 工具系统（Tool System）架构

## 5.1 V2 工具模型

核心文件：`packages/core/src/tool/tool.ts`

```typescript
const make: <Input, Output>(config: {
  description: string
  input: Input          // Effect Schema 定义输入
  output: Output        // Effect Schema 定义输出
  execute: (input, context: Tool.Context) => Effect.Effect<Output, ToolFailure>
  toModelOutput?: (input) => Tool.Content[]
}) => Definition<Input, Output>
```

**调用上下文**：
```typescript
interface Tool.Context {
  sessionID: Session.ID
  agent: Agent.ID
  assistantMessageID: Session.MessageID
  toolCallID: ToolCall.ID
}
```

## 5.2 内置工具清单

定义在 `packages/core/src/tool/builtins.ts`：

| 工具 | 功能 | 核心文件 |
|---|---|---|
| `read` | 读取文件，支持分页 | `read-filesystem.ts`, `read.ts` |
| `write` | 写入文件 | `write.ts` |
| `edit` | 编辑文件（精确字符串替换） | `edit.ts` |
| `grep` | 内容搜索 | `grep.ts` |
| `glob` | 文件模式匹配 | `glob.ts` |
| `bash` | 执行命令 | `bash.ts` |
| `websearch` | 网页搜索 | `websearch.ts` |
| `webfetch` | 网页获取 | `webfetch.ts` |
| `question` | 询问用户 | `question.ts` |
| `todowrite` | 创建管理 TODO | `todowrite.ts` |
| `skill` | 加载 skill | `skill.ts` |
| `apply_patch` | 应用补丁（新增/修改/删除） | `apply-patch.ts` |

## 5.3 注册与执行

**双层注册**：应用层 (ApplicationTools) + 位置层 (Tools.Service，优先级更高)

```text
Agent 请求 → LLM 解析 → 工具调用
  → ToolRegistry 查找 → 匹配注册名称
  → 权限检查 PermissionV2
  → 输入解码（Effect Schema）
  → 执行（Effect）
  → 输出编码 + Bound（大小限制）
  → 返回结果到 LLM
```

**Settlement 流程**：
```
1. 解析注册 → 找到有效的命名注册
2. 解码输入 → 如果无效，不执行工具
3. 调用工具 → 使用 runner 提供的上下文
4. 编码输出 → 如果无效，不返回成功
5. 投影输出 → toModelOutput 或结构化输出
6. 边界处理 → 过大内容截断到托管存储
```

## 5.4 输出边界处理

```typescript
// 通用输出边界
- 文本部分: 受限大小，超大的保存到托管存储
- 结构化元数据: 保留不变
- 原生媒体: 由生产者限制，保持原样
- 空内容: 度量结构化输出
```

## 5.5 工具注册示例

```typescript
// 注册一个自定义工具
tools.register({
  my_tool: Tool.make({
    description: "My custom tool",
    input: InputSchema,
    output: OutputSchema,
    execute: (input, context) =>
      Effect.gen(function* () {
        // 获取服务依赖
        const fs = yield* FileSystem.Service
        const permission = yield* PermissionV2.Service
        // 权限检查
        yield* permission.assert({ ... })
        // 执行
        return yield* fs.doSomething(input)
      }),
  }),
})
```

## 5.6 工具注册规则

- 同一名称最新注册生效
- 关闭注册只移除当前注册，暴露前一个
- Location 注册优先于应用注册
- 调用时捕获生效的注册，后续变更不影响运行中的调用
- 失效调用（注册被移除/替换后）被拒绝

---

# 第六篇: 会话系统（Session System）设计

## 6.1 V2 会话架构

```
Session:
├── id (ULID)
├── metadata (slug, title, agent, model, cost, tokens)
├── messages[]
│   ├── user (用户输入 + attachments)
│   ├── assistant (文本 + reasoning + tool_call)
│   └── tool_result (工具执行结果)
├── events[] (事件流)
│   ├── PromptAdmitted
│   ├── Prompted
│   ├── AgentSwitched
│   └── ModelSwitched
└── context_epoch (系统上下文快照)
```

## 6.2 V2 会话 API

核心文件：`packages/core/src/session.ts`

```typescript
interface Interface {
  create(input): Session
  get(sessionID): Session
  list(input): Session[]
  prompt(input): Admitted
  messages(input): Message[]
  context(sessionID): Message[]
  resume(sessionID): void
  interrupt(sessionID): void
  switchAgent(input): void
  switchModel(input): void
  compact(input): void
  shell(input): void
  skill(input): void
  events(input): Stream
}
```

## 6.3 上下文纪元（Context Epoch）

V2 的核心创新之一。每次 LLM 请求时，系统上下文（System Context）会被记录为一个 Context Epoch：

```text
第一次请求 → 初始化 Context Epoch
  ├── 环境事实（环境变量、日期）
  ├── 项目指令（AGENTS.md）
  ├── 配置指令（instructions）
  ├── Agent 技能提示（skills）
  └── 外部引用（references）

后续请求 → 对比变化
  ├── 无变化 → 复用基线
  ├── 有变化 → 生成系统消息 → 更新 Epoch 快照
  └── 自动压缩 → 重建完整基线

位置迁移 → 清除 Epoch，重新初始化基线
```

**Context Epoch 事件流**：
```text
Client    Runner    Context Registry    Epoch Store    History    LLM
  │         │             │                 │            │        │
  ├─ Admit ──────────────────────────────────────────────────────▶
  │         ├─ Observe context ─▶            │            │        │
  │         ◀─ Complete/unavail ─┤            │            │        │
  │         ├─ Init epoch ──────────────────▶│            │        │
  │         ├─ Promote input ────────────────────────────────────▶│
  │         ├─ Reconcile ──────▶            │            │        │
  │         ├─ Advance snapshot ────────────▶│            │        │
  │         ├─ Baseline + history ────────────────────────────────▶
```

## 6.4 自动压缩（Compaction）

当上下文接近模型限制时自动触发：

```text
请求前 → 估计完整请求大小
→ 超过模型窗口 - buffer
  → 启动压缩
  → 创建压缩检查点
  → 结构化摘要 + 保留最近上下文
  → 下次请求使用压缩后的基线

溢出触发 → Provider 拒绝（context overflow）
  → 自动再尝试一次压缩
  → 第二次溢出 → 终止（防止循环）
```

## 6.5 递送模式

| 模式 | 行为 |
|---|---|
| **steer** | 推进到下一个 provider turn 边界 |
| **queue** | 进入 FIFO 队列，当前 drain 结束后自动提升一个 |

## 6.6 事件驱动

Session 采用事件溯源：

```typescript
// 持久化事件类型
"session.next.compaction.started.1"   // 压缩开始
"session.next.compaction.ended.1"     // 压缩完成
PromptAdmitted                         // 输入被接收
Prompted                               // 输入已投递
AgentSwitched                          // Agent 切换
ModelSwitched                          // 模型切换
```

事件订阅支持：`sessions.events({ sessionID, after? })` — 从指定序列号后拉取事件。

---

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

# 第八篇: 插件系统（Plugin System）扩展开发

## 8.1 插件架构

核心文件：`packages/core/src/plugin.ts`

```typescript
interface Interface {
  add(id: ID, effect: Plugin["effect"]): void   // 热加载
  remove(id: ID): void                           // 热卸载
}
```

使用 Effect Scope 生命周期管理：
- `Scope.fork` 创建子作用域
- `Scope.close` 清理资源

## 8.2 插件钩子（Hooks）

文件：`packages/plugin/src/index.ts`

```typescript
interface Hooks {
  dispose?: () => void
  event?: (input) => void
  config?: (input: Config) => void
  tool?: Record<string, ToolDefinition>         // 自定义工具
  auth?: AuthHook                                // 自定义认证
  provider?: ProviderHook                        // Provider 扩展

  // 会话相关
  "chat.message"?: (input, output) => void
  "chat.params"?: (input, output) => void        // 修改 LLM 参数
  "chat.headers"?: (input, output) => void       // 修改请求头

  // 权限
  "permission.ask"?: (input, output) => void

  // 工具拦截
  "tool.execute.before"?: (input, output) => void // 调用前修改参数
  "tool.execute.after"?: (input, output) => void  // 调用后修改结果

  // 环境
  "shell.env"?: (input, output) => void           // 注入环境变量

  // 实验性
  "experimental.chat.messages.transform"?: (input, output) => void
  "experimental.chat.system.transform"?: (input, output) => void
  "experimental.provider.small_model"?: (input, output) => void
  "experimental.session.compacting"?: (input, output) => void
}
```

## 8.3 V2 Effect Plugin 系统

`packages/plugin/src/v2/effect/` 定义了基于 Effect 的 V2 插件：

```
v2/effect/
├── agent.ts         # Agent 操作
├── aisdk.ts         # AI SDK 集成
├── catalog.ts       # Provider/Catalog
├── command.ts       # 命令
├── context.ts       # 系统上下文
├── event.ts         # 事件
├── filesystem.ts    # 文件系统
├── integration.ts   # 集成
├── location.ts      # 位置
├── npm.ts           # NPM 包管理
├── path.ts          # 路径
├── plugin.ts        # 插件基础
├── reference.ts     # 引用
├── registration.ts  # 注册
└── skill.ts         # 技能
```

## 8.4 插件热加载生命周期

```text
Plugin.add(id, effect)
  → Scope.fork(scope) 创建子作用域
  → effect(host) 执行插件
  → 注册工具、钩子、Provider 等
  → Scope.close() 清理

Plugin.remove(id)
  → Scope.close(current) 关闭
  → 注册的变换自动回滚
```

**重载检测**：
```typescript
const locks = KeyedMutex.makeUnsafe<ID>()  // 防止并发加载
const loading = new Set<ID>()               // 加载中检测
// 如果 loading.has(id) → 检测到循环依赖 → 报错
```

---

# 第九篇: LLM 运行时与 Provider 集成

## 9.1 双层运行时架构

```text
Session Processor
    ↓
LLM.Service (llm.ts) — 会话层的 LLM 服务
    ↓
Native Gate — OPENCODE_EXPERIMENTAL_NATIVE_LLM=true
  ├── YES → native-runtime.ts
  │          → native-request.ts (构建 LLMRequest)
  │          → LLMClient / RequestExecutor
  └── NO  → AI SDK (默认)
             → streamText / generateText
             → ai-sdk.ts (适配为 LLMEvent)
    ↓
LLMEvent 流 (统一格式)
```

## 9.2 AI SDK 运行时

使用 Vercel AI SDK 作为默认路径：

```typescript
// 使用 streamText 进行流式请求
const result = streamText({ model, messages, tools, ... })
// ai-sdk.ts 将 fullStream 事件转换为统一的 LLMEvent
// 包括：text, reasoning, tool_call, tool_result, error
```

**结构化输出**：
```typescript
// 使用 generateObject 生成结构化数据（如 Agent 配置）
const result = generateObject({
  model,
  schema: GeneratedAgent,  // Effect Schema 转换
  messages: [system, user],
})
```

## 9.3 原生 LLM 运行时（试验性）

`@opencode-ai/llm` 包提供原生实现，支持：
- OpenAI 兼容 API
- Anthropic API
- 通过 `OPENCODE_EXPERIMENTAL_NATIVE_LLM=true` 启用

## 9.4 Provider 管理

**Provider 配置来源**（合并优先级从低到高）：
1. 内置 catalog 提供商
2. models.dev 远程数据（定时刷新）
3. 配置文件 provider 定义
4. 插件注册/provider hook
5. Auth 认证状态

**Provider 注册流程**：
```text
Config 文件 → ConfigProvider → Catalog.transform()
Plugin 激活 → Catalog.transform()
Auth 切换 → AuthPlugin → Catalog.transform()
models.dev → ModelsDevPlugin → Catalog.transform()
```

---

# 第十篇: 技术亮点与二次开发指南

## 10.1 架构亮点总结

1. **Event Sourcing + CQRS**: 会话完整事件溯源，支持回放和审计
2. **Effect 函数式编程**: 全栈 Effect v4，统一异步、DI、错误处理、并发
3. **Location 隔离**: 按工作目录隔离服务实例
4. **插件热加载**: 动态加载/卸载，无需重启
5. **Context Epoch**: 智能上下文管理，自动压缩
6. **Eager Tool Execution**: 主动工具执行，减少延迟
7. **双层 LLM 运行时**: AI SDK 默认 + 原生运行时 opt-in
8. **多层权限系统**: 默认权限 + Agent 权限 + 父会话继承 + 用户配置

## 10.2 值得关注的业务场景

1. **企业 AI 编程助手**: 完整的参考实现
2. **多 Agent 工作流引擎**: Orchestrator-Worker + 并行/串行委派
3. **可扩展工具平台**: 插件 SDK + 工具注册机制
4. **会话回溯与调试**: 事件溯源支持完整回放
5. **技能生态**: Skill 作为 AI 知识分发单元
6. **AI Agent 生成**: 让 LLM 自己编写 Agent 配置

## 10.3 二次开发关键文件

| 关注点 | 路径 |
|---|---|
| Agent 定义与配置 | `packages/opencode/src/agent/agent.ts` |
| Agent 核心模型 | `packages/core/src/agent.ts` |
| Subagent 权限 | `packages/opencode/src/agent/subagent-permissions.ts` |
| Agent 配置 Schema | `packages/core/src/config/agent.ts` |
| 工具系统 | `packages/core/src/tool/` (20 文件) |
| 会话管理 | `packages/core/src/session.ts` |
| 会话执行器 | `packages/core/src/session/runner/` |
| 插件 SDK | `packages/plugin/src/` |
| V2 插件 Effect | `packages/plugin/src/v2/effect/` (18 文件) |
| 技能系统 | `packages/core/src/skill.ts` |
| 背景任务 | `packages/core/src/background-job.ts` |
| 配置系统 | `packages/core/src/config.ts` |
| LLM 运行时 | `packages/opencode/src/session/llm/` (4 文件) |
| 原生 LLM | `packages/llm/` |
| V2 架构规格 | `specs/v2/` (10 文件) |
| 事件系统 | `packages/core/src/event.ts` |
| Provider 目录 | `packages/core/src/catalog.ts` |
| 插件钩子宿主 | `packages/core/src/plugin/host.ts` |

## 10.4 核心流程速查

### Agent 创建流程
```text
config → Config.Service → AgentV2.layer → 状态管理 → 查询
```

### Subagent 生成流程
```text
task() → 解析 Agent → 权限计算 → 创建会话 → 执行 → 返回
```

### 模型选择流程
```text
Session → Agent → Agent.model → Provider → Auth → API
```

### 插件加载流程
```text
config → PluginBoot → 安装 → 激活 → transform → 生效
```

## 10.5 系列学习路线

- **第一篇: 架构总览** → 建立全局认知
- **第二篇: Agent 系统** → 理解核心概念
- **第三篇: Subagent** → 掌握多 Agent 基础
- **第四篇: 多 Agent 协作** → 理解协作模式
- **第五篇: 工具系统** → 理解 Agent 能力边界
- **第六篇: 会话系统** → 深入运行机制
- **第七篇: 技能与命令** → 扩展 Agent 能力
- **第八篇: 插件系统** → 掌握扩展开发
- **第九篇: LLM 运行时** → 底层 AI 集成
- **第十篇: 最佳实践** → 总结与二次开发指南

## 10.6 关注的技术细节

1. **Effect 的优势与学习曲线**: 类型安全 + 依赖注入，但函数式编程有门槛
2. **双重 Schema 系统**: Effect Schema + Drizzle Schema + Config Schema
3. **权限系统的多层叠加**: 默认 → Agent → 父会话 → 用户
4. **V1 到 V2 的迁移**: 并行存在 `packages/core/src/v1/` 目录
5. **Durable vs In-memory**: 会话持久化 vs BackgroundJob 内存管理
6. **OpenTelemetry 集成**: 实验性功能，需要在配置中启用

---

## 📚 参考资源

- **代码仓库**: https://github.com/anomalyco/opencode
- **官方文档**: https://opencode.ai/docs
- **Effect v4**: https://effect.website
- **AI SDK**: https://sdk.vercel.ai
- **V2 设计文档**: `specs/v2/` 目录 (session.md, tools.md, instructions.md, config.md 等)
- **Drizzle ORM**: https://orm.drizzle.team
- **分析基准**: 2026年6月 dev 分支

---

*本系列文章通过对 OpenCode 源码的深度解析产出，重点聚焦 AI Agent 搭建、Subagent 创建、多智能体协作、工具系统、会话管理、技能/插件系统等核心主题。*


