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
