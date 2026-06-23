# 09 - LLM 运行时与 Provider 集成

## 编写 TODO

### 需要阅读的源码

- [ ] 深入阅读 `packages/core/src/session/runner/llm.ts`（380 行，完整读完，这是最核心的文件）
- [ ] 深入阅读 `packages/core/src/session/runner/model.ts`（207 行）
- [ ] 阅读 `packages/core/src/session/runner/max-steps.ts`（16 行）
- [ ] 阅读 `packages/core/src/session/runner/publish-llm-event.ts`
- [ ] 深入阅读 `packages/core/src/system-context/index.ts`（316 行）
- [ ] 阅读 `packages/core/src/session/context-epoch.ts`（174 行，与 system-context 配合）
- [ ] 阅读 `packages/core/src/session/compaction.ts`（246 行）
- [ ] 阅读 `packages/opencode/src/session/retry.ts`（201 行）
- [ ] 阅读 `packages/opencode/src/session/overflow.ts`（34 行）
- [ ] 阅读 `packages/llm/` 中的 AI SDK 适配代码

### 分析要点

- [ ] **Run Loop 数据流**: 
   ```
   runTurnAttempt
     → getSession → agents.select → loadSystemContext → ContextEpoch.initialize
     → models.resolve → SessionHistory.entriesForRunner
     → compactIfNeeded? (ContinueAfterCompaction)
     → isLastStep? (toolMaterialization = undefined + MAX_STEPS_PROMPT)
     → llm.stream(request) → forEach(LLMEvent):
         ├── providerError + isContextOverflowFailure + !hasAssistantStarted → overflowFailure
         ├── text/reasoning → publish
         ├── tool-call (local) → settle → publish → FiberSet.run
         └── tool-call (provider) → publish (needsContinuation = true)
     → providerStream exit:
         ├── overflowFailure + recoverOverflow → compactAfterOverflow (ContinueAfterOverflowCompaction)
         ├── LLMError + !hasProviderError → failAssistant
         ├── QuestionRejected → interrupt
         ├── tool fiber failure → failUnsettledTools
         └── success → needsContinuation → loop
   ```
- [ ] **TurnTransition 控制流**: 为什么用 `Effect.die(TurnTransitionError)` + `catchDefect`？
   - 因为 `runTurnAttempt` 在被 `Effect.scoped` 包围的内部（`return yield* Effect.uninterruptibleMask(...)`），普通返回值无法穿透 scope 边界
   - `die` 作为 "信号" 传播到 `catchDefect` 层，在那里识别并处理跳转
   - 这实际上是一种 "异常即控制流" 模式（类似 `goto`），但在 Effect 的类型系统下是安全的
- [ ] **Model 解析链**: 
   ```
   session.model → catalog.model.available() → find(providerID + modelID)
     → withVariant → produce(model, draft => ModelRequest.assign(draft.request, variant))
     → fromCatalogModel → Route.with({...}) → Model
   ```
- [ ] **SystemContext 的泛型 Source<A>**: 
   - 每个 source 有 5 个方法：`load`（加载数据）/ `baseline`（生成基线文本）/ `update`（生成更新文本）/ `removed`（生成移除文本）
   - `Snapshot` 持久化 source 的上次值，用于 reconcile 时的 diff 比较
   - `Generation` 是完整的基线文本 + 快照
- [ ] **溢出恢复安全限制**: 溢出恢复后再溢出 → die 报错（防止无限循环）

### 深度洞察

- `TurnTransitionError` + `Effect.die` 是实现"非局部跳转"的关键技巧。`runTurnAttempt` 在 `Effect.scoped` 内运行，拥有独立的资源生命周期。如果在 scope 内直接递归调用 `runTurn`，scope 不会释放——而 `die` 可以穿透 scoped 边界，让外层 `catchDefect` 重新构建整个请求。这是 Effect 中"跳出作用域"的惯用模式
- `runAfterOverflowCompaction` 和 `runTurn` 的区别是断熔器设计：溢出恢复后的尝试如果再次溢出，直接报错停止。这是为了防止"压缩-溢出-再压缩-再溢出"的死循环
- `isContextOverflowFailure` + `!hasAssistantStarted()` 的双条件检查很微妙：如果 LLM 已经输出了部分文本然后说"我溢出了"，不会触发压缩恢复——因为已经产生了语义，压缩会丢失这些文本。LLM 必须在"还没开始说"之前报告溢出才能触发恢复
- 模型解析中的 `produce(model, draft => ModelRequest.assign(draft.request, variant))` 使用 `immer` 做不可变更新——这说明 model 对象不可直接修改，必须通过 immer 的 draft 代理
- `promptCacheKey` 的设计：session ID 如果匹配 `ses_[0-9a-f]{64}` 格式，去掉 `ses_` 前缀作为 OpenAI 的 prompt cache key——因为 OpenAI 对 key 格式有特定要求

### 依赖

- [ ] 06 - 会话系统
- [ ] 02 - Agent 系统
