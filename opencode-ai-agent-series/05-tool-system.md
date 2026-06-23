# 05 - 工具系统架构

## 编写 TODO

### 需要阅读的源码

- [ ] 深入阅读 `packages/core/src/tool/tool.ts`（144 行，核心模型）
- [ ] 深入阅读 `packages/core/src/tool/registry.ts`（139 行，注册与执行）
- [ ] 阅读 `packages/core/src/tool/builtins.ts` — 列出所有内置工具
- [ ] 阅读 `packages/core/src/tool/application-tools.ts` — 应用层注册
- [ ] 深入阅读 1-2 个具体工具实现：
   - [ ] `packages/core/src/tool/read.ts` — 文件读取（带分页）
   - [ ] `packages/core/src/tool/edit.ts` — 精确替换编辑
   - [ ] `packages/core/src/tool/bash.ts` — 命令执行
   - [ ] `packages/core/src/tool/question.ts` — 询问用户
   - [ ] `packages/core/src/tool/apply-patch.ts` — 补丁应用（新增/修改/删除）
- [ ] 阅读 `packages/core/src/tool/tools.ts` — Tools.Service 定义
- [ ] 阅读 `packages/core/src/tool-output-store.ts` — 工具输出存储

### 分析要点

- [ ] **Tool.make 的实现细节**: 
   - 使用 `WeakMap<AnyTool, Runtime>` 分离"工具对象"和"运行时行为"
   - 工具对象本身是一个 `Object.freeze({})` 空对象——它只是 WeakMap 的 key
   - `definitions` Map 按名称缓存 ToolDefinition
- [ ] **双重 Schema 验证**: 
   - `Schema.decodeUnknownEffect(config.input)` — 验证 LLM 传入的参数
   - `Schema.encodeEffect(config.output)` — 验证工具返回的结果格式
   - 输入验证失败 → `"Invalid tool input: ${error.message}"`
   - 输出验证失败 → `"Tool returned an invalid value for its output schema: ${error.message}"`
- [ ] **双层注册**: ApplicationTools（内置）→ Tools.Service（运行时/插件注册，覆盖优先级）
- [ ] **Stale Tool Call 检测**: materialize() 创建 identity → 注册 → 执行时比较 identity
- [ ] **权限过滤**: whollyDisabled() —— `findLast` 规则匹配，如果 `resource: "*"` 且 `effect: "deny"` 则移除工具
- [ ] **工具名验证**: `[A-Za-z][A-Za-z0-9_-]{0,63}` —— 64 字符限制，禁止点（.），禁止特殊字符
- [ ] **Settlement 流程**: 执行 → 输出 bound（截断/托管存储） → 返回结果到 LLM
- [ ] **toModelOutput**: 工具如何将结构化输出转换为 LLM 可读的文本/文件内容

### 深度洞察

- WeakMap + `Object.freeze({})` 的设计几乎是"反面向对象"的：工具定义不携带任何方法，所有行为在 Runtime 中。好处是工具可以安全地深度冻结、跨进程传递、序列化。但代价是失去了 TypeScript 的实例类型检查——`AnyTool` 本质上是一个"无类型的 key"
- 工具名限定了长度和字符集——这是为了满足 LLM 的 tool calling 约束（OpenAI / Anthropic 对工具名都有字符限制）。但不允许点号（`.`）意味着 `fs.read` 这样的命名空间名称不被允许，所有工具名是扁平的
- Effect Schema 的 `Schema.toJsonSchemaDocument(schema)` + `$defs` 合并是"Effect Schema → LLM JSON Schema"的桥接。这意味着定义 Effect Schema 的 tool input 可以自动生成 LLM 能理解的 JSON Schema——不需要重复定义

### 依赖

- [ ] 02 - Agent 系统
- [ ] 04 - 多智能体协作（BackgroundJob 的 token-based 陈旧检测）
