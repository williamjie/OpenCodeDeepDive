# OpenCode 模型稳定性工程深度解析 — 系列文章

> 生成时间: 2026-06-23 09:16
> 文件名: 20260623-0916-opencode-model-instability-analysis.md
> 生成模型: deepseek-v4-flash-free

---

## 📋 概览

| 项目 | 信息 |
|---|---|
| **系列主题** | AI Agent 的 "80% 水下工程" — 模型不稳定性处理 |
| **核心问题** | LLM 输出不可靠、API 故障、JSON 异常、上下文溢出、权限误判 |
| **覆盖范围** | 重试策略、JSON Schema 验证、上下文窗口管理、权限守卫、工具防护、步骤限制、运行时恢复 |
| **代码包** | `packages/core`, `packages/opencode`, `packages/llm` |

> **背景**：在与不可预测的 LLM （幻觉、JSON 不稳定、API 超时、Token 耗尽等）打交道时，AI Agent 真正需要 80% 精力处理的是这些"水下工程"。OpenCode 在这方面构建了一套系统性的防御体系。

---

## 📑 文章索引

| 编号 | 文章标题 | 核心内容 |
|---|---|---|
| 一 | 重试与回退：应对 LLM API 不可靠 | 智能重试策略、错误分类、退避算法、Rate Limit 处理、免费额度 upsell |
| 二 | JSON Schema 验证：驯服模型输出 | Effect Schema 类型安全、结构化输出验证、解码/解析模式 |
| 三 | 上下文窗口管理：防止 Agent 记忆溢出 | Token 预估、溢出检测、自动压缩、LLM 辅助摘要生成 |
| 四 | 权限系统：防止模型误操作 | Ruleset 规则评估、Ask/Allow/Deny 模型、持久化权限记忆 |
| 五 | 工具系统防护：无效调用与陈旧调用 | Stale Tool Call 检测、工具定义验证、执行失败处理 |
| 六 | 运行时恢复：全方位容错机制 | Provider Error 处理、Tool Fiber 管理、中断恢复、Question Rejection |
| 七 | 设计哲学与工程实践 | 综合回顾：防御深度、Effect 函数式错误处理、可观测性 |

---

# 第一篇: 重试与回退 — 应对 LLM API 不可靠

## 1.1 问题背景

LLM API 是现代 AI Agent 的命脉，却也是最不可靠的环节：

- **服务器故障**：5xx 错误、超时
- **限流（Rate Limit）**：429 Too Many Requests
- **额度耗尽**：免费层级/付费账户余额不足
- **过载**：Provider overloaded
- **偶发错误**：连接重置、DNS 解析失败

OpenCode 在 `packages/opencode/src/session/retry.ts` 中实现了一套智能的重试系统。

## 1.2 核心参数

```typescript
export const RETRY_INITIAL_DELAY = 2000           // 初始延迟 2s
export const RETRY_BACKOFF_FACTOR = 2             // 指数退避因子 2x
export const RETRY_MAX_DELAY_NO_HEADERS = 30_000  // 30s 上限（无头信息时）
export const RETRY_MAX_DELAY = 2_147_483_647      // 32-bit signed int 上限
```

## 1.3 延迟计算：从 API 响应到精确等待

`delay()` 函数是重试延迟计算的核心，它有三种模式：

### 模式 1：服务端指定 retry-after-ms（精确毫秒）
```typescript
const retryAfterMs = headers["retry-after-ms"]
if (retryAfterMs) {
  const parsedMs = Number.parseFloat(retryAfterMs)
  if (!Number.isNaN(parsedMs)) return cap(parsedMs)
}
```
最高优先级，直接使用服务端指定的毫秒等待时间。

### 模式 2：服务端指定 retry-after（秒/HTTP 日期）
```typescript
const retryAfter = headers["retry-after"]
if (retryAfter) {
  // 尝试解析为秒数
  const parsedSeconds = Number.parseFloat(retryAfter)
  if (!Number.isNaN(parsedSeconds)) return cap(Math.ceil(parsedSeconds * 1000))
  // 尝试解析为 HTTP 日期格式
  const parsed = Date.parse(retryAfter) - Date.now()
  if (!Number.isNaN(parsed) && parsed > 0) return cap(Math.ceil(parsed))
}
```
同时兼容 `Retry-After: 120`（秒数）和 `Retry-After: Wed, 21 Oct 2026 07:28:00 GMT`（HTTP 日期）两种格式。

### 模式 3：指数退避（无头信息时）
```typescript
return cap(Math.min(
  RETRY_INITIAL_DELAY * Math.pow(RETRY_BACKOFF_FACTOR, attempt - 1),
  RETRY_MAX_DELAY_NO_HEADERS
))
```
当没有服务端头信息时，使用指数退避：2s → 4s → 8s → 16s → 30s（封顶）。

## 1.4 错误分类引擎：什么值得重试？

`retryable()` 函数判断一个错误是否可重试，它遵循精确的分类逻辑：

```
错误输入
  │
  ├── ContextOverflowError ───────────────→ ❌ 不重试（上下文溢出是设计问题，不是临时故障）
  │
  ├── APIError（5xx / 可重试标记）
  │     │
  │     ├── FreeUsageLimitError ──────────→ 🔴 不可重试，但提供 UpSell 引导
  │     ├── GoUsageLimitError ────────────→ 🔴 不可重试，但显示恢复时间
  │     └── Overloaded / 普通 APIError ───→ ✅ 可重试
  │
  ├── 文本错误匹配
  │     ├── "rate increased too quickly" ─→ ✅ 可重试
  │     ├── "rate limit" ─────────────────→ ✅ 可重试
  │     └── "too many requests" ──────────→ ✅ 可重试
  │
  └── JSON 错误 body 解析
        ├── too_many_requests ────────────→ ✅ 可重试
        ├── exhausted / unavailable ──────→ ✅ 可重试（Provider 过载）
        └── rate_limit ───────────────────→ ✅ 可重试
```

### 关键设计决策：ContextOverflowError 不重试

```typescript
// context overflow errors should not be retried
if (SessionV1.ContextOverflowError.isInstance(error)) return undefined
```
这是一个重要的架构决定：上下文溢出是 token 预算耗尽的问题，重试只会浪费更多 token。正确的做法是通过压缩（Compaction）而非重试来恢复。

### 5xx 自动重试策略

```typescript
// 5xx errors are transient server failures and should always be retried,
// even when the provider SDK doesn't explicitly mark them as retryable.
if (!error.data.isRetryable && !(status !== undefined && status >= 500)) return undefined
```
即使 Provider SDK 没有标记某个错误为可重试的，只要状态码 >= 500，就强制重试。这是一个"宁可重试也不要静默失败"的设计哲学。

## 1.5 限流 UpSell：业务与工程的无缝衔接

OpenCode 在检测到免费额度耗尽时，不是简单地报错，而是触发一套 UpSell 流程：

```typescript
return {
  message: GO_UPSELL_MESSAGE,
  action: {
    reason: "free_tier_limit",
    provider,
    title: "Free limit reached",
    message: "Subscribe to OpenCode Go for reliable access...",
    label: "subscribe",
    link: GO_UPSELL_URL,
  },
}
```

对于 Go 付费用户，还提供更精细的额度使用信息：

```typescript
return {
  message: `${message} - ${link}`,
  action: {
    reason: "account_rate_limit",
    provider,
    title: "Go limit reached",
    message: `... It will reset in ${resetIn}. To continue using this model now...`,
    label: "open settings",
    link,
  },
}
```

这种设计让 AI Agent 不仅是一个工具，还能在业务层面提供引导。

## 1.6 Effect Schedule 集成

`policy()` 函数是整个重试策略与 Effect 函数式编程的桥梁：

```typescript
export function policy(opts: {
  provider: string
  parse: (error: unknown) => Err
  set: (input: { attempt: number; message: string; action?: Retryable["action"]; next: number }) => Effect.Effect<void>
}) {
  return Schedule.fromStepWithMetadata(
    Effect.succeed((meta: Schedule.InputMetadata<unknown>) => {
      const error = opts.parse(meta.input)
      const retry = retryable(error, opts.provider)
      if (!retry) return Cause.done(meta.attempt)  // 不可重试 → 停止调度
      return Effect.gen(function* () {
        const wait = delay(meta.attempt, ...)
        const now = yield* Clock.currentTimeMillis
        yield* opts.set({ attempt, message, action, next: now + wait })  // 发布重试状态
        return [meta.attempt, Duration.millis(wait)]  // 返回延迟时间
      })
    }),
  )
}
```

**设计亮点**：Effect 的 `Schedule` 结合 `Cause.done()` 来终止重试序列，当 `retryable()` 返回 `undefined` 时，重试自动停止。这构成了一个类型安全的重试调度器。

---

# 第二篇: JSON Schema 验证 — 驯服模型输出

## 2.1 问题背景

LLM 的本质是概率生成器，其输出天然不稳定：

- JSON 格式错误（缺少括号、逗号错位）
- 字段缺失或类型错误
- 产生幻觉字段（hallucinated properties）
- 工具参数超出预期范围

## 2.2 Effect Schema：类型安全的验证体系

OpenCode 大量使用 Effect Schema（`@effect/schema`）来对 LLM 生成的输出进行运行时验证。Schema 模式将 TypeScript 的编译时类型系统延伸到运行时。

### 典型模式：Schema.Class + Schema.TaggedErrorClass

```typescript
// 定义结构化的错误类型
export class ModelNotSelectedError extends Schema.TaggedErrorClass<ModelNotSelectedError>()(
  "SessionRunnerModel.ModelNotSelectedError",
  { sessionID: SessionSchema.ID },
) {}

export class ModelUnavailableError extends Schema.TaggedErrorClass<ModelUnavailableError>()(
  "SessionRunnerModel.ModelUnavailableError",
  { providerID: ProviderV2.ID, modelID: ModelV2.ID },
) {}
```

**TaggedErrorClass** 确保错误可以被 Effect 的 `Cause` 系统精确匹配：
- 通过 `_tag` 字段（类名自动生成）实现模式匹配
- Effect 的 `Effect.catchTag()` 可以根据 `_tag` 精确捕获特定错误

### 数据模型 Schema 验证

```typescript
// 会话 ID 验证
export const ID = Schema.String.pipe(
  Schema.startsWith("ses"),        // 必须以 "ses" 开头
  Schema.brand("SessionV2.ID"),    // 品牌化类型
  withStatics((schema) => ({
    create: (id?: string) => schema.make(id ?? "ses_" + Identifier.ascending()),
  })),
)

// 权限请求 Schema
export const Request = Schema.Struct({
  id: ID,
  sessionID: SessionV2.ID,
  action: Schema.String,
  resources: Schema.Array(Schema.String),
  save: Schema.Array(Schema.String).pipe(Schema.optional),
  metadata: Schema.Record(Schema.String, Schema.Unknown).pipe(Schema.optional),
  source: Source.pipe(Schema.optional),
}).annotate({ identifier: "PermissionV2.Request" })
```

### Schema 解码的核心 API

Effect Schema 提供多个解码函数：

```typescript
// 严格解码：失败 → Error
Schema.decodeUnknown(schema)(data)
Schema.decodeUnknownEither(schema)(data)  // Either<ParseError, T>
Schema.decodeUnknownPromise(schema)(data)  // Promise<T>

// 异步解码（支持表单式验证）
Schema.decodeUnknownAsync(schema)(data)
Schema.decodeUnknownEffect(schema)(data)   // Effect<T, ParseError>

// 可选解码与容错
Schema.decodeUnknownOption(schema)(data)   // Option<T>
Schema.parse(schema)(data)                 // 同时解析和转换
Schema.asserts(schema)(data)               // 类型守卫
```

### 工具输入验证链

当 LLM 产生工具调用时，验证链如下：

```
LLM Output (text/JSON)
    │
    ▼
Schema.decodeUnknown (类型检查 + 结构验证)
    │
    ├── 成功 → 执行工具
    │
    └── 失败 → ParseError
          │
          ├── 缺失字段 → 返回错误给 LLM
          ├── 类型错误 → 返回错误给 LLM
          └── 多余字段 → 可配置忽略/报错
```

这种机制保证：即使 LLM 产生格式错误的工具调用，系统也不会崩溃，而是将错误信息反馈给 LLM 让其自我修正。

## 2.3 Permission Schema 验证体系

权限系统有自己的 Schema 层：

```typescript
export class RejectedError extends Schema.TaggedErrorClass<RejectedError>()(
  "PermissionV2.RejectedError", {}
) {}

export class CorrectedError extends Schema.TaggedErrorClass<CorrectedError>()(
  "PermissionV2.CorrectedError",
  { feedback: Schema.String },  // 包含用户反馈
) {}

export class DeniedError extends Schema.TaggedErrorClass<DeniedError>()(
  "PermissionV2.DeniedError",
  { rules: PermissionSchema.Ruleset },  // 显示被什么规则拒绝
) {}
```

**三层错误设计**：
- `DeniedError`：权限规则明确拒绝（确定性）
- `RejectedError`：用户拒绝（人为决策）
- `CorrectedError`：用户修改后拒绝（带有反馈信息，可用于 LLM 自我修正）

## 2.4 工具输出验证

工具执行结果也需要 Schema 验证：

```typescript
export const AskResult = Schema.Struct({
  id: ID,
  effect: PermissionSchema.Effect,  // "allow" | "deny" | "ask"
}).annotate({ identifier: "PermissionV2.AskResult" })
```

工具注册时，OpenCode 对工具名称进行验证：

```typescript
yield* Effect.forEach(entries, ([name]) => validateName(name), { discard: true })
```

---

# 第三篇: 上下文窗口管理 — 防止 Agent 记忆溢出

## 3.1 问题背景

LLM 有固定的上下文窗口限制（通常 8K-200K tokens）。当 Agent 持续运行时：
- 历史对话不断累积
- 工具执行结果占用大量 token
- 编码后的文件内容占比极大
- LLM 在处理长上下文时质量下降（Lost in the Middle）

OpenCode 设计了多层的上下文窗口管理系统。

## 3.2 三层架构

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: 溢出检测 (overflow.ts)                         │
│ "是否需要做什么？"                                       │
├─────────────────────────────────────────────────────────┤
│ Layer 2: 可用 Token 计算 (overflow.ts)                   │
│ "还剩多少空间？"                                         │
├─────────────────────────────────────────────────────────┤
│ Layer 3: 自动压缩 (compaction.ts)                        │
│ "如何腾出空间？"                                         │
└─────────────────────────────────────────────────────────┘
```

## 3.3 可用 Token 计算

`usable()` 函数计算当前可用的输入 token 数量：

```typescript
const COMPACTION_BUFFER = 20_000  // 保留 20K token 给输出

export function usable(input: { cfg: ConfigV1.Info; model: Provider.Model; outputTokenMax?: number }) {
  const context = input.model.limit.context
  if (context === 0) return 0

  const reserved =
    input.cfg.compaction?.reserved ??
    Math.min(COMPACTION_BUFFER, ProviderTransform.maxOutputTokens(input.model, input.outputTokenMax))
  
  return input.model.limit.input
    ? Math.max(0, input.model.limit.input - reserved)     // 有 input_limit → 减去保留
    : Math.max(0, context - ProviderTransform.maxOutputTokens(input.model, input.outputTokenMax))  // 无 input_limit → 总上下文减去输出预算
}
```

**关键设计**：
- 始终为模型输出保留 token 空间（避免输出被截断）
- 支持两种限流模式：`model.limit.input`（输入限制）和 `model.limit.context`（总上下文限制）
- 预留 token 可配置（`cfg.compaction?.reserved`）

## 3.4 溢出检测

`isOverflow()` 决定是否已经溢出：

```typescript
export function isOverflow(input: { ... }) {
  if (input.cfg.compaction?.auto === false) return false  // 自动压缩关闭 → 不过问
  if (input.model.limit.context === 0) return false        // 无限上下文模型

  const count =
    input.tokens.total || input.tokens.input + input.tokens.output + input.tokens.cache.read + input.tokens.cache.write
  return count >= usable(input)  // 当前 token >= 可用 token → 溢出
}
```

**精确计量**：`count` 包含了 prompt caching 的读写 token，这使得估算更加准确。

## 3.5 自动压缩引擎

`compaction.ts` 实现了整个压缩流程：

### 压缩触发点

压缩有两个触发时机：

1. `compactIfNeeded()` — 在每次 LLM 请求前检查是否需要压缩
2. `compactAfterOverflow()` — 在 LLM 返回上下文溢出错误后被动压缩

### 压缩配置

```typescript
const settings = (documents: readonly Config.Entry[]) => ({
  auto: true,          // 默认开启自动压缩
  buffer: 20_000,      // 保留 20K token buffer
  tokens: 8_000,        // 压缩后保留 8K token 的最新历史
})
```

### 核心算法：`select()`

`select()` 函数实现"保留最新 N tokens"的逻辑：

```typescript
const select = (entries: readonly Entry[], tokens: number) => {
  // 1. 过滤掉已存在的压缩消息，只保留真实对话
  const conversation = entries
    .filter((entry) => entry.message.type !== "compaction")
    .map((entry) => serialize(entry.message))

  // 2. 从后往前累加 token，找到在 tokens 预算内的分割点
  let total = 0
  let split = conversation.length
  for (let index = conversation.length - 1; index >= 0; index--) {
    const next = total + Token.estimate(conversation[index])
    if (next > tokens) {
      // 在超出的消息上精确分割
      const remaining = Math.max(0, tokens - total) * 4
      if (remaining > 0) {
        splitPrefix = conversation[index].slice(0, -remaining)
        splitSuffix = conversation[index].slice(-remaining)
        split = index + 1
      }
      break
    }
    total = next
    split = index
  }

  // 3. 返回"历史部分"和"最新部分"
  return {
    head:      [...conversation.slice(0, split), splitPrefix].join("\n\n"),  // 需要被总结的部分
    recent:    [splitSuffix, ...conversation.slice(split)].join("\n\n"),     // 保留的原始内容
  }
}
```

**这是一个"总结历史 + 保留最新"的策略**，其灵感来自于"滑动窗口 + 摘要"模式。

### 让 LLM 总结 LLM

OpenCode 使用 LLM 自身来生成摘要，而不是使用简单的文本截断：

```typescript
const SUMMARY_TEMPLATE = `Output exactly the Markdown structure shown inside <template>...
<template>
## Goal
- [single-sentence task summary]

## Constraints & Preferences
- [user constraints, preferences, specs, or "(none)"]

## Progress
### Done
- [completed work or "(none)"]
### In Progress
- [current work or "(none)"]
### Blocked
- [blockers or "(none)"]

## Key Decisions
- [decision and why, or "(none)"]

## Next Steps
- [ordered next actions or "(none)"]

## Critical Context
- [important technical facts, errors, open questions, or "(none)"]

## Relevant Files
- [file or path: why it matters, or "(none)"]
</template>

Rules:
- Keep every section, even when empty.
- Use terse bullets, not prose paragraphs.
- Preserve exact file paths, commands, error strings, and identifiers when known.
- Do not mention the summary process or that context was compacted.`
```

**设计决策**：
- **结构化模板**：8 个固定段落确保摘要的完整性和一致性
- **禁止提及压缩过程**：避免让 LLM 在回复中意外暴露"系统提示"（越狱保护）
- **保留精确引用**：文件路径、错误信息、命令等关键信息必须原样保留
- **合并而非替换**：当已有摘要时，LLM 被要求"更新"而不是"重写"

### 压缩执行流程

```typescript
const compactAfterOverflow = Effect.fn("SessionCompaction.compactAfterOverflow")(function* (input: Input) {
  // 1. 安全检查：模型必须有上下文限制
  if (context === undefined || context <= 0) return false

  // 2. 计算分割
  const selected = select(input.entries, config.tokens)

  // 3. 获取前次摘要（如果存在）
  const previousSummary = input.entries.find((entry) => entry.message.type === "compaction")?.message

  // 4. 构建提示词（更新模式 vs 创建模式）
  const summaryPrompt = buildPrompt({
    previousSummary: previousSummary?.type === "compaction" ? previousSummary.summary : undefined,
    context: [previousSummary?.type === "compaction" ? previousSummary.recent : "", selected.head],
  })

  // 5. 调用 LLM 生成摘要（通过 stream 方式，处理 provider 错误）
  const summarized = yield* dependencies.llm.stream(
    LLM.request({ model: input.model, messages: [Message.user(summaryPrompt)], tools: [], generation: { maxTokens: summaryOutput } }),
  ).pipe(Stream.runForEach((event) => { ... }))

  // 6. 如果压缩失败，回退到不压缩（不阻塞主流程）
  if (!summarized || failed || !summary.trim()) return false

  // 7. 发布压缩事件（可观测性）
  yield* dependencies.events.publish(SessionEvent.Compaction.Ended, { ... })
  return true
})
```

### 预检机制（节省 token）

`compactIfNeeded()` 在发送请求前先估算 token 用量：

```typescript
if (estimate({ system, messages, tools }) <= context - Math.max(output, config.buffer))
  return false  // 还够用，跳过压缩
```

**token 估算**使用 `Token.estimate()` 函数：
```typescript
const estimate = (value: unknown) => Token.estimate(JSON.stringify(value))
```
简单的 JSON 序列化后估算，不需要调用外部 tokenizer。

## 3.6 工具输出截断

为了防止工具输出消耗过多 token，`truncate()` 函数将工具输出截断到 2000 字符：

```typescript
const TOOL_OUTPUT_MAX_CHARS = 2_000
const truncate = (value: string) =>
  value.length <= TOOL_OUTPUT_MAX_CHARS ? value : `${value.slice(0, TOOL_OUTPUT_MAX_CHARS)}\n[truncated]`
```

在序列化时：
```typescript
[Tool result]: ${truncate(serializeToolContent(part.state.content))}
```

---

# 第四篇: 权限系统 — 防止模型误操作

## 4.1 问题背景

LLM 可能会：
- 调用不该调用的工具（如删除文件）
- 读取敏感信息（如密码文件）
- 执行破坏性操作（如大规模删除）
- 在错误的时间执行正确的操作

OpenCode 的 Permission V2 系统在 `packages/core/src/permission.ts` 中实现。

## 4.2 三层权限评估模型

```
用户输入 Action + Resources
        │
        ▼
  ┌─────────────────┐
  │ Layer 1: Agent   │  Agent 级别的权限规则（内置规则）
  │ 权限规则         │  └── 默认：{"action": "*", "resource": "*", "effect": "deny"}
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ Layer 2: 用户    │  用户保存的权限规则（持久化）
  │ 保存规则         │  └── 保存在 permission_saved 表中
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ Layer 3: 评估    │  Wildcard 匹配 + 优先级合并
  │ 引擎             │  └── findLast 规则 → 最终效果
  └────────┬────────┘
           │
     ┌─────┴──────┬──────┐
     ▼            ▼      ▼
   Allow        Ask    Deny
   (允许)      (询问)   (拒绝)
```

## 4.3 Rule → Effect 映射

每个权限规则由三个字段定义：

```typescript
type Rule = {
  action: string    // 动作模式（支持 wildcard）
  resource: string  // 资源模式（支持 wildcard）
  effect: Effect    // "allow" | "deny" | "ask"
}
```

## 4.4 核心评估引擎

`evaluate()` 函数是权限判断的核心：

```typescript
export function evaluate(action: string, resource: string, ...rulesets: Ruleset[]): Rule {
  return (
    rulesets
      .flat()                                                       // 合并所有规则集
      .findLast((rule) =>                                           // 找到最后匹配的规则
        Wildcard.match(action, rule.action) && Wildcard.match(resource, rule.resource)
      ) ?? {
      action,
      resource: "*",
      effect: "ask",                                                // 默认：询问用户
    }
  )
}
```

**关键设计决策**：
- **`findLast`（而非 `findFirst`）**：后面的规则覆盖前面的，方便在配置中追加覆盖规则
- **Wildcard 匹配**：`*` 匹配所有，支持 `fs.read.*` 等模式
- **默认 `ask`**：没有匹配规则时，询问用户而不是静默允许或拒绝

## 4.5 Deny 优先原则

```typescript
const evaluateInput = Effect.fnUntraced(function* (input: AssertInput) {
  const rules = yield* configured(input.sessionID, input.agent)

  // 首先检查是否有 deny 规则（Deny 优先）
  if (denied(input, rules)) return { effect: "deny" as const, rules }

  // 然后合并保存的规则
  const all = [...rules, ...(yield* savedRules())]
  const effects = input.resources.map((resource) =>
    evaluate(input.action, resource, all).effect
  )
  // 三个资源中任何一个 deny → deny；任何一个 ask → ask；全 allow → allow
  const effect: Effect = effects.includes("deny") ? "deny" : effects.includes("ask") ? "ask" : "allow"
  return { effect, rules: all }
})
```

**Deny 优先于 Allow**：即使某些资源 allow，只要一个资源 deny，整个请求就被拒绝。这是安全设计中的"最小权限原则"。

## 4.6 异步 Ask 流程

当权限评估结果为 "ask" 时，系统进入异步等待用户批准的模式：

```typescript
const create = (request: Request, agent?: AgentV2.ID) =>
  EffectRuntime.uninterruptible(
    EffectRuntime.gen(function* () {
      const deferred = yield* Deferred.make<void, RejectedError | CorrectedError>()
      // 存储到 pending map 中
      pending.set(request.id, { request, agent, deferred })
      // 发布 Ask 事件（触发 UI 显示）
      yield* events.publish(Event.Asked, request)
      return { request, agent, deferred }
    }),
  )
```

用户通过 `reply()` 进行响应：

```typescript
const reply = EffectRuntime.fn("PermissionV2.reply")((input: ReplyInput) =>
  // "reject" → 拒绝（可包含反馈）
  // "always" → 允许并记住
  // "once"   → 仅本次允许（不持久化）
)
```

### 级联拒绝

当用户拒绝一个请求时，同 session 中所有 pending 请求也被级联拒绝：

```typescript
if (input.reply === "reject") {
  // 拒绝当前请求
  yield* Deferred.fail(existing.deferred, ...)
  pending.delete(input.requestID)
  // 级联拒绝同 session 的所有 pending 请求
  for (const [id, item] of pending) {
    if (item.request.sessionID !== existing.request.sessionID) continue
    yield* Deferred.fail(item.deferred, new RejectedError())
    pending.delete(id)
  }
}
```

### 智能 Auto-Allow

当用户选择 "always" 时，系统不仅保存当前规则，还会自动批准后续匹配的相同请求：

```typescript
if (input.reply === "always" && existing.request.save?.length) {
  yield* saved.add({ projectID, action, resources })
}
// ... 然后扫描同 session 的 pending 请求，如果符合新规则则自动批准
```

## 4.7 错误类型体系

Permission 系统定义了精确的错误层次：

| 错误类型 | Tag | 含义 | 恢复策略 |
|---|---|---|---|
| `RejectedError` | `PermissionV2.RejectedError` | 用户拒绝 | 尝试其他方案 |
| `CorrectedError` | `PermissionV2.CorrectedError` | 用户拒绝并给出反馈 | 根据反馈修正后重试 |
| `DeniedError` | `PermissionV2.DeniedError` | 规则拒绝 | 不可恢复 |

---

# 第五篇: 工具系统防护 — 无效调用与陈旧调用

## 5.1 问题背景

LLM 生成的工具调用可能：
- **Stale（陈旧）**：LLM 在切换上下文后，仍然调用上一次 materialize 的工具
- **Unknown（未知）**：LLM 幻觉出一个不存在的工具
- **Permission Denied（权限不足）**：LLM 试图调用被禁止的工具

## 5.2 Stale Tool Call 检测

在 `ToolRegistry.materialize()` 中，每个工具注册时都有一个唯一的身份令牌：

```typescript
const settleWith = Effect.fn("ToolRegistry.settle")(function* (input: ExecuteInput, advertised?: object) {
  const registration = local.get(input.call.name)?.at(-1)?.registration ?? applications.entries().get(input.call.name)
  if (!registration)
    return { result: { type: "error", value: advertised ? `Stale tool call: ${input.call.name}` : `Unknown tool: ${input.call.name}` } }

  // 身份检查：advertised（当前 materialization）vs registration（注册时的 identity）
  if (advertised && registration.identity !== advertised)
    return { result: { type: "error", value: `Stale tool call: ${input.call.name}` } }
  // ...
})
```

**工作原理**：
1. 每次 `materialize()` 创建一个新的空对象 `{}` 作为身份标识
2. 工具定义（definitions）通过闭包捕获这个身份标识
3. 工具执行时，LLM 传回的 `advertised` 与 registry 中的 `registration.identity` 比较
4. 如果不一致 → Stale Tool Call → 返回错误

```
materialize() ──→ 创建 identity = {}
                     │
                     ▼
                注册到 definitions（捕获 identity）
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    LLM 调用    重新 materialize   LLM 调用
    identity=①    → identity=②    identity=①
        │                         │
        ▼                         ▼
    匹配 ✓ 执行             不匹配 ✓ 返回 Stale Error
```

## 5.3 工具注册生命周期

工具注册使用 Effect 的 `Scope` 和 `addFinalizer` 实现自动清理：

```typescript
const register = Effect.fn("ToolRegistry.register")(function* (tools) {
  yield* Effect.uninterruptible(
    Effect.gen(function* () {
      const token = {}
      for (const [name, tool] of entries)
        local.set(name, [...(local.get(name) ?? []), { token, registration: { identity: {}, tool } }])

      // Scope 结束时自动清理（基于 Effect 的 addFinalizer）
      yield* Effect.addFinalizer(() =>
        Effect.sync(() => {
          for (const [name] of entries) {
            const registrations = local.get(name)?.filter((r) => r.token !== token) ?? []
            if (registrations.length > 0) local.set(name, registrations)
            else local.delete(name)
          }
        }),
      )
    }),
  )
})
```

**设计亮点**：
- 使用 `Effect.Scope` 实现自动注册/注销
- 多个注册者同名工具时使用栈结构（`.at(-1)`）
- 在 Scope 清理时自动恢复前一个注册

## 5.4 工具执行失败的类型安全处理

当工具执行失败时，通过 `Effect.catchTag` 精确匹配 LLM 错误：

```typescript
const pending = yield* settle(registration.tool, input.call, {
  sessionID: input.sessionID,
  agent: input.agent,
  assistantMessageID: input.assistantMessageID,
  toolCallID: input.call.id,
}).pipe(
  Effect.map((output) => ({ output })),
  Effect.catchTag("LLM.ToolFailure", (failure) =>
    Effect.succeed({ result: { type: "error" as const, value: failure.message } }),
  ),
)
```

## 5.5 工具权限过滤

在 `materialize()` 阶段，会对工具进行权限过滤：

```typescript
for (const [name, registration] of registrations)
  if (whollyDisabled(permission(registration.tool, name), permissions))
    registrations.delete(name)
```

`whollyDisabled()` 检查工具的 action 是否被完全禁止：

```typescript
function whollyDisabled(action: string, rules: PermissionV2.Ruleset) {
  const rule = rules.findLast((rule) => Wildcard.match(action, rule.action))
  return rule?.resource === "*" && rule.effect === "deny"
}
```

---

# 第六篇: 运行时恢复 — 全方位容错机制

## 6.1 问题背景

Agent 运行时可能遇到各种异常：
- 网络中断
- 模型返回错误
- 上下文溢出
- 工具执行失败
- 用户中断会话
- 问题（Question）拒绝导致循环中断

## 6.2 LLM 运行时架构

`packages/core/src/session/runner/llm.ts` 实现了完整的 LLM 会话运行器。整个 run 循环如下：

```
run Turn Loop
   │
   ├── runTurnAttempt (单次尝试)
   │    │
   │    ├── check context overflow → compactIfNeeded → ContinueAfterCompaction
   │    │
   │    ├── llm.stream(request) 主请求
   │    │    │
   │    │    ├── providerError → 上下文溢出 → compactAfterOverflow → ContinueAfterOverflowCompaction
   │    │    ├── providerError → 普通错误 → failAssistant + 返回错误
   │    │    ├── tool-call (local) → settle + 继续
   │    │    └── tool-call (provider-executed) → 记录结果 + 继续
   │    │
   │    ├── tool fiber 等待
   │    │    ├── 成功 → 继续
   │    │    ├── 中断/Interrupt → 清理 + 中断循环
   │    │    ├── Question Rejected → 清理 + 中断循环
   │    │    └── 失败 → failUnsettledTools
   │    │
   │    └── 是否 needContinuation?
   │         ├── 是 → 继续 runTurn (step + 1)
   │         └── 否 → 检查是否有 pending steer/queue
   │              ├── 有 → 继续
   │              └── 无 → 结束 run 循环
   │
   └── TurnTransition 控制流
        ├── ContinueAfterCompaction → 从头构建请求，重试
        ├── ContinueAfterOverflowCompaction → 特殊路径，限制一次恢复
        └── 其他 defect → die（作为异常传播）
```

## 6.3 上下文溢出恢复：多层保护

### 预检恢复（主动保护）

```typescript
// 在发送请求前，先检查是否需要压缩
if (yield* compaction.compactIfNeeded({ sessionID, entries, model, request }))
  return yield* Effect.die(continueAfterCompaction(currentStep))
```
如果预测到会溢出，则先压缩，然后通过 `TurnTransition` 跳转到新的循环。

### 后置恢复（被动保护）

```typescript
if (
  recoverOverflow &&
  !publisher.hasAssistantStarted() &&
  isContextOverflowFailure(overflowFailure ?? failure) &&
  (yield* restore(recoverOverflow({ sessionID, entries, model, request })))
)
  return yield* Effect.die(continueAfterOverflowCompaction(currentStep))
```
如果 Provider 返回上下文溢出错误（且 Assistant 尚未开始输出），进行溢出恢复压缩。

### 限制递归：溢出恢复不会再次溢出

```typescript
const runAfterOverflowCompaction: RunTurn = Effect.fnUntraced(function* (sessionID, promotion, step) {
  return yield* runTurnAttempt(sessionID, promotion, step).pipe(
    Effect.catchDefect(
      Effect.fnUntraced(function* (defect) {
        if (!(defect instanceof TurnTransitionError)) return yield* Effect.die(defect)
        if (defect.transition._tag === "ContinueAfterOverflowCompaction")
          return yield* Effect.die("Post-compaction provider attempt cannot recover another overflow")
        // ...
      }),
    ),
  )
})
```

**关键安全措施**：溢出恢复后的尝试如果仍然溢出，不会再次递归，而是直接 die 报错，防止无限循环。

## 6.4 Max Steps 保护机制

`max-steps.ts` 定义了 Agent 步骤限制的边界：

```typescript
export const MAX_STEPS_PROMPT = `CRITICAL - MAXIMUM STEPS REACHED
The maximum number of steps allowed for this task has been reached.
Tools are disabled until next user input. Respond with text only.
STRICT REQUIREMENTS:
1. Do NOT make any tool calls
2. MUST provide a text response summarizing work done so far
...
Any attempt to use tools is a critical violation. Respond with text ONLY.`
```

在 runner 中的应用：

```typescript
const isLastStep = agent.info?.steps !== undefined && currentStep >= agent.info.steps
const toolMaterialization = isLastStep ? undefined : yield* tools.materialize(agent.info?.permissions)
```

**两种武器**：
1. 当达到最后步骤时，不 materialize 工具（`toolMaterialization = undefined`），LLM 无法调用工具
2. 插入 `MAX_STEPS_PROMPT` 作为系统级指令，明确要求 LLM 只返回文本

## 6.5 工具 Fiber 管理

工具执行使用 Effect FiberSet 进行并发管理：

```typescript
const toolFibers = yield* FiberSet.make<void, ToolOutputStore.Error>()
// ... 每个工具调用注册到 FiberSet
return yield* Effect.uninterruptibleMask((restore) =>
  Effect.gen(function* () {
    // ... provider stream 流程
    const settled = yield* restore(awaitToolFibers(toolFibers)).pipe(Effect.exit)
    // 处理各种完成/中断情况
  }),
)
```

### 中断处理矩阵

```
Stream 状态 \ Tool Fiber 状态 →  成功     中断(Interrupt)  失败(Error)
─────────────────────────────────────────────────────────────────
Success                          继续     清理+中断         failUnsettledTools
Failure (Provider Error)         继续     清理+中断         failUnsettledTools
Failure (Interrupt)              清理     清理+中断         清理+中断
```

### 中断工具清理

当会话中断时，未完成的工具会被标记为"失败"：

```typescript
const failInterruptedTools = Effect.fn("SessionRunner.failInterruptedTools")(function* (sessionID) {
  for (const message of yield* getContext(sessionID)) {
    if (message.type !== "assistant") continue
    for (const tool of message.content) {
      if (tool.type !== "tool" || (tool.state.status !== "pending" && tool.state.status !== "running")) continue
      yield* events.publish(SessionEvent.Tool.Failed, {
        sessionID, timestamp: yield* DateTime.now,
        assistantMessageID: message.id, callID: tool.id,
        error: { type: "unknown", message: "Tool execution interrupted" },
      })
    }
  }
})
```

## 6.6 双重 TurnTransition 控制流

OpenCode 使用 Effect 的 `Effect.die` + `catchDefect` 实现函数式的非局部跳转：

```typescript
// 定义跳转型
class TurnTransitionError extends Error {
  constructor(readonly transition: TurnTransition) { super() }
}

// 发起点：需要压缩时
if (yield* compaction.compactIfNeeded({...}))
  return yield* Effect.die(continueAfterCompaction(currentStep))

// 捕获点：在 runTurn 中
const runTurn: RunTurn = Effect.fnUntraced(function* (sessionID, promotion, step) {
  return yield* runTurnAttempt(sessionID, promotion, step, compaction.compactAfterOverflow).pipe(
    Effect.catchDefect(
      Effect.fnUntraced(function* (defect) {
        if (!(defect instanceof TurnTransitionError)) return yield* Effect.die(defect)
        yield* Effect.yieldNow  // 让出执行，防止栈溢出
        if (defect.transition._tag === "ContinueAfterOverflowCompaction")
          return yield* runAfterOverflowCompaction(...)
        return yield* runTurn(sessionID, undefined, defect.transition.step)
      }),
    ),
  )
})
```

**为什么不用简单的递归调用？**
- 将非正常的控制流（需要重新构建请求）与正常返回分离
- `catchDefect` 可以在任意深层嵌套中捕获跳转请求
- `Effect.yieldNow` 防止递归导致的栈溢出

## 6.7 Question Rejected 处理

当用户拒绝一个 Question 时，Effect 的 `Cause` 会被捕获：

```typescript
// 检测 Question 拒绝
const isQuestionRejected = (cause: Cause.Cause<unknown>) =>
  cause.reasons.some((reason) => Cause.isDieReason(reason) && reason.defect instanceof QuestionV2.RejectedError)

// 处理
if (settled._tag === "Failure" && isQuestionRejected(settled.cause)) {
  yield* FiberSet.clear(toolFibers)
  yield* withPublication(publisher.failUnsettledTools("Tool execution interrupted"))
  return yield* Effect.interrupt  // 中断主循环
}
```

**设计意图**：用户拒绝 Question 后，LLM 不应该"看到"这个拒绝结果并尝试不同的方式。整个会话应该立即停止。

---

# 第七篇: 设计哲学与工程实践

## 7.1 总体防御深度

```
                    ┌─────────────────────────┐
                    │    业务层防御            │
                    │  UpSell、额度管理、模型选择 │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │    运行时层防御          │
                    │  Retry、Max Steps、Fiber  │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │    压缩层防御            │
                    │  Overflow、Compaction    │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │    Schema 层防御         │
                    │  Effect Schema 验证      │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │    权限层防御            │
                    │  Permission Ruleset      │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │    工具层防御            │
                    │  Stale Detection、Name   │
                    │  Validation              │
                    └─────────────────────────┘
```

## 7.2 Effect 函数式编程的稳定工程优势

OpenCode 使用 Effect（v4）作为核心编程范式，这在稳定性方面提供了独特优势：

| 特性 | 稳定性贡献 | 示例 |
|---|---|---|
| **类型安全的错误** | 错误类型编译时检查，没有 "undefined is not a function" | `Schema.TaggedErrorClass` |
| **精确的错误捕获** | 通过 `_tag` 模式匹配，不会误捕或漏捕 | `Effect.catchTag("PermissionV2.RejectedError")` |
| **Scope 管理** | 资源生命周期自动管理，没有资源泄漏 | `Effect.addFinalizer` → 工具注册自动清理 |
| **Fiber 并发** | 结构化并发，不会产生孤儿 goroutine | `FiberSet.run()` 自动管理工具 fibers |
| **Cause 系统** | 完整记录错误链，不丢失任何上下文 | `Cause.findErrorOption` + `Cause.hasInterrupts` |
| **不可变数据** | 状态变化可控，没有并发竞态 | `produce(model, draft => ...)` imer 不可变更新 |

## 7.3 关键模式总结

### 模式：分层错误恢复（以上下文溢出为例）

```
Level 1: 预检（compactIfNeeded）→ 主动压缩 → 继续
Level 2: 溢出检测（isContextOverflowFailure）→ 被动压缩 → 继续
Level 3: 恢复失败（溢出恢复后再次溢出）→ die 报错 → 不可恢复
```

### 模式：TurnTransition 非局部跳转

```
需要重新构建请求 → Effect.die(TurnTransitionError) → catchDefect → 重新构建
```

### 模式：Deny 优先评估

```
遍历所有规则 → 如果任何规则 deny → 立即拒绝
否则 → 遍历资源 → 存在 ask → 询问；全 allow → 允许
```

### 模式：智能的压缩摘要

```
对话历史 → 分割为 historical + recent
historical → LLM 生成结构化摘要
recent → 保持原始可读
下次压缩 → 合并历史摘要 + 最新历史 → 生成新摘要
```

## 7.4 可观测性事件

每个决策点都发布了结构化事件，支持监控和调试：

| 事件 | 触发点 | 包含信息 |
|---|---|---|
| `Compaction.Started` | 压缩开始 | sessionID, messageID, reason |
| `Compaction.Ended` | 压缩完成 | sessionID, summary, recent |
| `Tool.Failed` | 工具执行中断 | sessionID, callID, error type |
| `permission.v2.asked` | 权限询问 | requestID, action, resources |
| `permission.v2.replied` | 权限回复 | requestID, reply effect |

---

## 对比总结

相比简单的 LLM API 调用封装，OpenCode 的模型稳定性工程体现了以下设计理念：

1. **失败是常态，容错是默认**：从 retry 策略到压缩恢复，系统默认所有操作都可能失败
2. **精准而非粗暴**：错误分类不是简单的"可重试/不可重试"，而是根据 Provider、状态码、错误消息模式进行精确分类
3. **LLM 辅助 LLM**：用 LLM 生成摘要，用 Effect Schema 验证 LLM 输出，构建自修复的反馈循环
4. **安全深度防御**：权限系统 + 工具 stale 检测 + deno 优先评估，多层防止模型误操作
5. **业务与工程融合**：限流 UpSell 将技术错误转化为业务增长机会

---

*本系列文章基于 OpenCode 源码分析，版本参考 master 分支。所有代码引用均在 `packages/core` 和 `packages/opencode` 包中。*
