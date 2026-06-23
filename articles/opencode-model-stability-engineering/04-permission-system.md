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
