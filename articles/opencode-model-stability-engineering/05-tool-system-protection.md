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
