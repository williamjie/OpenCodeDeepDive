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
