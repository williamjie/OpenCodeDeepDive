# 第九篇: LLM 运行时与 Provider 集成

## 9.1 双层运行时架构

```text
Session Processor
    ↓
LLM.Service (llm.ts) — 会话层的 LLM 服务
    ↓
Native Gate — OPENCODE_EXPERIMENTAL_NATIVE_LLM=true
  ├── YES → native-runtime.ts
  │          → native-request.ts (构建 LLMRequest)
  │          → LLMClient / RequestExecutor
  └── NO  → AI SDK (默认)
             → streamText / generateText
             → ai-sdk.ts (适配为 LLMEvent)
    ↓
LLMEvent 流 (统一格式)
```

## 9.2 AI SDK 运行时

使用 Vercel AI SDK 作为默认路径：

```typescript
// 使用 streamText 进行流式请求
const result = streamText({ model, messages, tools, ... })
// ai-sdk.ts 将 fullStream 事件转换为统一的 LLMEvent
// 包括：text, reasoning, tool_call, tool_result, error
```

**结构化输出**：
```typescript
// 使用 generateObject 生成结构化数据（如 Agent 配置）
const result = generateObject({
  model,
  schema: GeneratedAgent,  // Effect Schema 转换
  messages: [system, user],
})
```

## 9.3 原生 LLM 运行时（试验性）

`@opencode-ai/llm` 包提供原生实现，支持：
- OpenAI 兼容 API
- Anthropic API
- 通过 `OPENCODE_EXPERIMENTAL_NATIVE_LLM=true` 启用

## 9.4 Provider 管理

**Provider 配置来源**（合并优先级从低到高）：
1. 内置 catalog 提供商
2. models.dev 远程数据（定时刷新）
3. 配置文件 provider 定义
4. 插件注册/provider hook
5. Auth 认证状态

**Provider 注册流程**：
```text
Config 文件 → ConfigProvider → Catalog.transform()
Plugin 激活 → Catalog.transform()
Auth 切换 → AuthPlugin → Catalog.transform()
models.dev → ModelsDevPlugin → Catalog.transform()
```

---
