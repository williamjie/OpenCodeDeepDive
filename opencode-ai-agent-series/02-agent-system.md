# 02 - AI Agent 系统设计与实现

## 编写 TODO

### 需要阅读的源码

- [ ] 深入阅读 `packages/core/src/agent.ts`，理解 `AgentV2.Info` 的 Schema 定义
- [ ] 深入阅读 `packages/opencode/src/agent/agent.ts`，分析 7 个内置 Agent 的完整配置（重点：各自用了哪些权限规则）
- [ ] 阅读 `packages/core/src/config/agent.ts`，理解 Agent 配置的 Schema 定义和覆盖规则
- [ ] 阅读 `packages/opencode/src/agent/prompt/` 中的系统提示词文件
- [ ] 阅读 `packages/core/src/config.ts`，理解 Config Service 如何加载 agent 配置

### 分析要点

- [ ] **AgentV2.Info 与应用层 Info 的区别**: Core 层定义的是最小领域模型（id / model / request / system / mode / permissions），应用层扩展了运行时字段（name / temperature / topP / color / variant / prompt / options / steps）。为什么 split？
- [ ] **7 个内置 Agent 的差异化设计**:
   - `build`（全放开，`question: allow`, `plan_enter: allow`）——为什么 build 允许 `plan_enter`？
   - `plan`（禁止编辑工具，允许 `task.general`）——plan 模式下仍然可以创建 subagent 做代码搜索
   - `explore`（仅允许 read / grep / glob / bash / webfetch / websearch / list——为什么 bash 被允许？因为是"只读 bash"，没有写权限）
   - `general`（除 `todowrite` 外全放开）——为什么禁止 todowrite？
   - `compaction` / `title` / `summary`（`*: deny`）——这 3 个隐藏 Agent 没有工具可调用，为什么还需要被定义为 Agent？
- [ ] **权限覆盖规则**: `Permission.merge(defaults, {...}, userRules)`——三组规则的合并逻辑
- [ ] **Agent 选择逻辑**: `select(id?)` — 指定 ID → 配置默认 → build → 第一个可见 primary
- [ ] **用户自定义 Agent**: 配置如何从 `opencode.jsonc` 加载并合并到内置 Agent？

### 深度洞察

- 隐藏 Agent（compaction / title / summary）权限全部 `"*": "deny"`，说明它们只是"系统提示词容器"。它们的运行路径不经过 Tool Registry，执行结果直接从 LLM 文本流处理
- `plan` Agent 禁止所有 `edit.*` 但允许 `task.general`——这是一个"只读审查 + 可委派研究"的架构
- `steps` 字段达到限制时，`isLastStep` 会导致 `toolMaterialization = undefined`，LLM 无法再调用工具。这是防无限循环的"最后防线"
- Agent 的 `mode` 字段（primary / subagent / all）决定了它在 UI 中的可见性和 `task()` 工具的可选性

### 依赖

- [ ] 01 - 架构总览
