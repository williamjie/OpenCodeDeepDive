# 第六篇: 运行时恢复 — 全方位容错机制

> **用人话说**：Agent 运行时可能遇到各种异常，OpenCode 有一套"兜底"机制，确保不会轻易崩溃。

---

## 为什么需要运行时恢复？

Agent 运行时可能遇到各种异常：

| 异常 | 用人话说 |
|---|---|
| 网络中断 | 断网了 |
| 模型返回错误 | AI 报错了 |
| 上下文溢出 | 记忆装不下了 |
| 工具执行失败 | 工具坏了 |
| 用户中断会话 | 用户不想干了 |
| 问题（Question）拒绝 | 用户拒绝了某个问题 |

---

## LLM 运行时架构

整个 run 循环如下：

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

---

## 上下文溢出恢复：多层保护

### 预检恢复（主动保护）

```
发送请求前 → 检查是否需要压缩 → 先压缩 → 继续
```

**用人饭店**：就像出门前先检查行李——如果超重了先整理，而不是到了机场再整理。

### 后置恢复（被动保护）

```
LLM 返回溢出错误 → 被动压缩 → 继续
```

**用人饭店**：就像到了机场才发现超重——只能现场整理了。

### 限制递归：溢出恢复不会再次溢出

```
溢出恢复后的尝试如果仍然溢出 → 不会再次递归 → 直接报错
```

**用人饭店**：就像"不过三"——整理一次还超重，就不能再整理了，直接放弃。

---

## Max Steps 保护机制

防止 Agent 无限循环：

```
达到最大步骤数 → 禁用工具 → 只返回文本 → 强制结束
```

### 两种武器

1. **不 materialize 工具**：LLM 无法调用工具
2. **插入 MAX_STEPS_PROMPT**：明确要求 LLM 只返回文本

**用人饭店**：就像给员工设了"加班上限"——到了上限就不能干活了，只能汇报进度。

---

## 工具 Fiber 管理

工具执行使用 Effect FiberSet 进行并发管理：

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

```
会话中断 → 扫描未完成的工具 → 标记为"Tool execution interrupted"
```

**用人饭店**：就像项目取消时，正在做的任务也要"收尾"——不能半途而废。

---

## 双重 TurnTransition 控制流

OpenCode 使用 Effect 的 `Effect.die` + `catchDefect` 实现函数式的非局部跳转：

```
需要重新构建请求 → Effect.die(TurnTransitionError) → catchDefect → 重新构建
```

### 为什么不用简单的递归调用？

| 原因 | 用人饭店 |
|---|---|
| 将非正常的控制流与正常返回分离 | 把"特殊操作"和"普通返回"分开 |
| `catchDefect` 可以在任意深层嵌套中捕获跳转请求 | 不管嵌套多深都能捕获 |
| `Effect.yieldNow` 防止递归导致的栈溢出 | 防止"卡死" |

**用人饭店**：就像"紧急出口"——不管你在几楼，都能直接跳到一楼。

---

## Question Rejected 处理

当用户拒绝一个 Question 时：

```
用户拒绝 → 清理工具 → 中断主循环
```

**设计意图**：用户拒绝 Question 后，LLM 不应该"看到"这个拒绝结果并尝试不同的方式。整个会话应该立即停止。

**用人饭店**：就像用户说"不干了"——立即停止，不要试图说服用户。

---

## 一句话总结

运行时恢复通过多层保护（预检、后置、递归限制）、Max Steps 保护、Fiber 管理、TurnTransition 控制流，确保 Agent 在各种异常情况下都能"兜底"。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-model-stability-engineering/06-runtime-recovery.md)*
