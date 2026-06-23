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
