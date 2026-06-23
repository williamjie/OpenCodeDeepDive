# 第九篇: LLM 运行时与 Provider 集成

> **用人话说**：AI 大脑怎么接进来。LLM 运行时是 Agent 与 AI 模型之间的"翻译官"。

---

## 双层运行时架构

OpenCode 有两套 LLM 运行时：

```
Session Processor
    ↓
LLM.Service — 会话层的 LLM 服务
    ↓
Native Gate — 是否启用原生运行时？
  ├── YES → 原生运行时（实验性）
  └── NO  → AI SDK（默认）
    ↓
LLMEvent 流（统一格式）
```

**用人饭店**：就像手机有两个处理器——默认用省电的，需要高性能时切换到旗舰的。

---

## AI SDK 运行时（默认）

使用 Vercel AI SDK 作为默认路径：

| 功能 | 用人话说 |
|---|---|
| `streamText` | 流式请求（边生成边返回） |
| `generateObject` | 生成结构化数据（如 Agent 配置） |

**用人饭店**：就像和 AI 聊天——你问一句，它边想边答，不是等想完了再一起说。

### 结构化输出

LLM 生成的内容需要符合特定格式（Schema 验证）：

```
LLM 输出 → Schema 验证 → 符合格式 → 使用
                          不符合格式 → 报错/重试
```

**用人饭店**：就像填表格——必须按格式填，填错了系统会提示你。

---

## 原生 LLM 运行时（试验性）

`@opencode-ai/llm` 包提供原生实现：

| 特性 | 用人话说 |
|---|---|
| OpenAI 兼容 API | 支持 OpenAI 格式 |
| Anthropic API | 支持 Anthropic 格式 |
| 需要手动启用 | `OPENCODE_EXPERIMENTAL_NATIVE_LLM=true` |

**用人饭店**：就像换了个引擎——性能可能更好，但还在测试阶段。

---

## Provider 管理

### Provider 是什么？

Provider 是 AI 模型的提供者，比如 OpenAI、Anthropic、Google。

### 配置来源（优先级从低到高）

| 优先级 | 来源 | 用人话说 |
|---|---|---|
| 1 | 内置 catalog | 系统自带的 |
| 2 | models.dev 远程数据 | 远程更新的 |
| 3 | 配置文件 provider | 用户配置的 |
| 4 | 插件注册/provider hook | 插件注册的 |
| 5 | Auth 认证状态 | 认证信息 |

**用人饭店**：就像手机运营商——系统自带的、远程更新的、用户配置的、插件注册的，优先级越来越高。

### 注册流程

```
Config 文件 → ConfigProvider → Catalog.transform()
Plugin 激活 → Catalog.transform()
Auth 切换 → AuthPlugin → Catalog.transform()
models.dev → ModelsDevPlugin → Catalog.transform()
```

**用人饭店**：就像多个来源的联系人合并——系统自带的、用户添加的、插件注册的，最后合并成一个完整的列表。

---

## 一句话总结

LLM 运行时是 Agent 与 AI 模型之间的"翻译官"，支持 AI SDK（默认）和原生运行时（实验性），Provider 管理支持多来源配置合并。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-ai-agent-architecture/09-llm-runtime.md)*
