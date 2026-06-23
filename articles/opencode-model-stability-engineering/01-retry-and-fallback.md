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
