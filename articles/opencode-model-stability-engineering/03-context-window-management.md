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
