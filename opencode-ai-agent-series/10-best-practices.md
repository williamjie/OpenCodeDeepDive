# 10 - 技术亮点与最佳实践

## 编写 TODO

### 需要阅读的源码

- [ ] 通读 `specs/v2/session.md` — V2 会话架构文档
- [ ] 通读 `specs/v2/tools.md` — V2 工具设计文档
- [ ] 通读 `specs/v2/config.md` — V2 配置设计文档
- [ ] 回顾前 9 篇文章的所有关键发现
- [ ] 阅读 `packages/core/src/event.ts` — 事件系统基础
- [ ] 快速浏览 `packages/core/src/v1/` — V1 遗留代码

### 分析要点

- [ ] **Event Sourcing + CQRS**: 
   - 所有会话状态变更为持久化事件流
   - 事件可回放（replayable transforms）
   - 但 BackgroundJob 明确选择"不持久化"
   - 这种"混合策略"的取舍是什么？哪些状态需要事件溯源，哪些不需要？
- [ ] **Location 隔离**: 
   - 按工作目录隔离服务实例
   - `Location.Service` + `LocationServiceMap` 实现多目录独立状态
   - 这是"轻量级多租户"——每个工作目录是一个租户
   - 代价：跨目录的 Agent 通信/会话共享变得困难
- [ ] **Token-based Stale Detection**:
   - 两处使用：BackgroundJob + ToolRegistry
   - `token: object` 空对象引用比较
   - vs 数字版本号：更轻量（无需生成序列号）、不溢出
   - 局限：只回答"是当前这个吗"，不回答"这是第几次"
- [ ] **Effect 函数式编程的权衡**:
   - 优势：类型安全、错误处理（Cause 系统）、并发（Fiber）、DI（Context/Service）
   - 代价：类型复杂度爆炸（`Effect<A, E, R>` 的三个泛型）、学习曲线陡峭、调试困难（不可变状态的调用栈）
- [ ] **迁移路径 V1 → V2**: 并行存在的 `v1/` 目录，两套 Session 和 Permission 系统

### 深度洞察

- OpenCode 最大的架构智慧可能在于 **"分层到极致"的哲学**：Core 层只定义 Schema 和接口，不给实现——实现全部在 opencode 层或插件层。这使得 Core 层非常稳定，变动都在外围
- Effect 的采用是一把双刃剑：它为并发 AI Agent 提供了类型安全的"轨道"（railway-oriented programming），但团队成员需要深入理解 Effect 的范式才能有效开发。这可能是一个"技术选型即团队筛选"的例子
- 多个系统的"写时复制"模式（`immer produce`、`SynchronizedRef.modify`、`new Map(jobs).set()`）贯穿整个代码库——这是 Effect 的"不可变性"哲学在架构层面的体现
- `specs/v2/` 目录中的设计文档（session.md / tools.md / config.md）展示了"文档驱动开发"的实践——先写设计规格，再写实现

### 与第二系列的联系

本文是第二系列（模型稳定性工程：`20260623-0916-opencode-model-instability-analysis.md`）的前置知识。建议在阅读第二系列前先掌握：

- Permission V2 的评估模型（allow / deny / ask）
- Compaction 引擎的触发机制（compactIfNeeded / compactAfterOverflow）
- Runner 的 TurnTransition 控制流（die + catchDefect）
- Retry 策略的延迟计算（retry-after-ms / retry-after / 指数退避）

### 依赖

- [ ] 01-09 所有文章
