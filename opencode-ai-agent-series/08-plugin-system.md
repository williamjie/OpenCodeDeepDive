# 08 - 插件系统扩展开发

## 编写 TODO

### 需要阅读的源码

- [ ] 深入阅读 `packages/plugin/src/index.ts` — 插件 SDK 完整 Hooks 定义（335 行）
- [ ] 阅读 `packages/core/src/plugin.ts` — 插件宿主
- [ ] 阅读 `packages/plugin/src/v2/effect/` 目录下的 18 个文件，分类理解
- [ ] 阅读 `packages/plugin/src/example.ts` — 插件示例
- [ ] 阅读 `packages/plugin/src/tool.ts` — 工具插件定义
- [ ] 阅读 `packages/plugin/src/tui.ts` — TUI 插件
- [ ] 阅读 `packages/plugin/src/shell.ts` — Shell 插件

### 分析要点

- [ ] **插件生命周期**: `add(id, effect)` → Scope.fork → `effect(host)` → 返回 Hooks → Scope.close 清理
- [ ] **20+ 钩子分类**:
   - **会话**: `chat.message` / `chat.params` / `chat.headers`
   - **工具**: `tool.execute.before` / `tool.execute.after` / `tool.definition`
   - **权限**: `permission.ask`
   - **环境**: `shell.env`
   - **实验性**: `chat.messages.transform` / `chat.system.transform` / `provider.small_model` / `session.compacting` / `compaction.autocontinue` / `text.complete`
- [ ] **AuthHook**: OAuth 流程的完整生命周期（prompts → authorize → callback → token）
- [ ] **ProviderHook**: 动态注册模型 Provider。`models(provider, ctx) => Record<string, ModelV2>`
- [ ] **V2 Effect 插件**: 18 个文件覆盖核心系统的每一个方面——agent、catalog、command、context、event、filesystem、integration、location、npm、path、plugin、reference、registration、skill
- [ ] **热加载与回滚**: KeyedMutex 防止并发加载，loading Set 检测循环依赖，Scope 自动回滚

### 深度洞察

- `tool.definition` 钩子允许插件拦截工具的定义——修改 `description` 和 `parameters`。这意味着插件可以"改写工具的自我介绍"，改变 LLM 对工具的理解。比如一个安全插件可以将 `bash` 工具的 description 改为"小心使用，将在沙箱中执行"
- `experimental.session.compacting` 钩子返回 `{ context: string[], prompt?: string }`：`context` 是追加的上下文行，`prompt` 是完全替换压缩提示词。这个"追加 vs 替换"的二分法非常实用——简单的插件只需追加，复杂的插件可以完全接管压缩逻辑
- `experimental.provider.small_model` 钩子很有意思：它允许插件注入一个"小模型"用于简单任务。这暗示 OpenCode 有计划做"任务路由"——根据任务复杂度自动选择模型，而不是所有请求都用同一个模型
- 实验性钩子的大量存在（`experimental.` 前缀）说明 OpenCode 还在快速迭代中。这些 API 未来可能变动或移除

### 依赖

- [ ] 05 - 工具系统
- [ ] 06 - 会话系统
