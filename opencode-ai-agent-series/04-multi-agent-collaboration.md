# 04 - 多智能体协作机制

## 编写 TODO

### 需要阅读的源码

- [ ] 深入阅读 `packages/core/src/background-job.ts`（364 行，完整读完）
- [ ] 阅读 `packages/core/src/session/run-coordinator.ts`，理解 SessionRunCoordinator
- [ ] 阅读 `packages/core/src/session/runner/llm.ts` 中的工具 fiber 管理部分（`FiberSet.run()` / `awaitToolFibers()`）
- [ ] 找到 `task` 工具定义，理解它如何创建 subagent 会话
- [ ] 阅读 `packages/core/src/config.ts` 中的 config transform 机制（插件热加载如何触发重配置）

### 分析要点

- [ ] **Orchestrator-Worker 模式**: 主 Agent 作为 Orchestrator，subagent 作为 Worker。Orchestrator 负责分解任务（"我需要搜索 A 和 B，你们同时去查"）和合成结果
- [ ] **BackgroundJob 的并发模型**:
   - 所有状态在 `SynchronizedRef<Map<string, Active>>` 中
   - 每次修改都 `new Map(jobs).set(id, next)`（写时复制）
   - `Deferred` 实现 completion/promotion 通知
   - `Scope.close()` 实现生命周期清理
   - `token: object` 实现陈旧检测
- [ ] **extend() 链式延续**: `Deferred.await(previous)` → input.run → `Deferred.succeed(tail)`。为什么需要这种链式设计？
- [ ] **Eager Tool Execution**: V2 runner 的核心优化——LLM 还在流式输出时，已经解析到的工具调用并发执行
- [ ] **Background → Foreground**: `promote()` 如何将后台 job 提升到前台？`waitForPromotion()` 的等待机制

### 深度洞察

- `SynchronizedRef.modify` + 不可变 Map 是 Effect 中的"事务性状态修改"模式：回调函数中的 `jobs` 是当前状态的快照，修改是在快照上创建新 Map，然后原子化替换。这比 `synchronized` 或 `Mutex.lock()` 更安全——回调中不会发生死锁
- `token: object` 模式：空对象引用比较（`===`）作为版本号。每次 start() 创建新的 `{}`，settle 时比较 token 是否一致。这种方式比数字版本号更轻量（无需生成和比较数字），且不会溢出。但它的局限是"只确保是最新，不确保是历史第几个"——token 只回答"是我期待的这次吗"，不回答"是第几次"
- BackgroundJob 声明"非持久化"——这是一个关键设计决策。持久化的 job 状态需要单独的 "durable ownership slice"。这种分阶段实现避免了"一次性建完所有功能"的风险
- Prompt cache key 设计：`ses_[0-9a-f]{64}` 格式的 session ID 会被裁剪为纯 hex 字符串作为 OpenAI 的 prompt cache key——因为 OpenAI 要求的 cache key 不能带 `ses_` 前缀

### 依赖

- [ ] 03 - Subagent 机制
