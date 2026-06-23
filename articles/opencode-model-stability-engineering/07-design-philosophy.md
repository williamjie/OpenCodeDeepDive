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
