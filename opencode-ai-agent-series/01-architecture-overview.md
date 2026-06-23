# 01 - OpenCode 架构总览与核心技术栈

## 编写 TODO

### 需要阅读的源码

- [ ] 浏览 `packages/core/src/` 目录结构，理解核心层（agent / config / event / permission / session / system-context / tool）的包间依赖
- [ ] 浏览 `packages/opencode/src/` 目录结构，理解应用层（agent / provider / session / config / tool）的组织方式
- [ ] 阅读 `specs/v2/instructions.md`，提取 V2 五大设计原则（Core 薄层、插件化、可重放变换、事件溯源、Location 作用域）
- [ ] 快速扫描 root `package.json`，理解 monorepo 的 25+ 包依赖关系
- [ ] 找一份 `opencode.jsonc` 示例配置，理解多级配置机制（全局 → 项目 → .opencode/）
- [ ] 浏览 `packages/core/src/location.ts`，理解 Location 服务的作用

### 分析要点

- [ ] **Monorepo 分层逻辑**: Core 层（领域 Schema / 接口 / 事件）→ OpenCode 层（应用编排 / Provider 适配）→ Plugin 层（扩展）→ LLM 层（原生运行时）。为什么 AI SDK（Vercel AI SDK）在默认路径，而 `@opencode-ai/llm` 作为 opt-in？
- [ ] **Effect v4 的侵入性**: 不仅仅是 `async/await` 替代品——`Context.Service`实现 DI、`Layer`管理生命周期、`Scope`管理资源、`Schema`做运行时验证。对比传统的 Express/NestJS 模式，Effect 为 AI Agent 场景带来了什么？
- [ ] **双重 Schema**: Effect Schema（运行时验证）vs Drizzle Schema（ORM 持久化）vs Config Schema（配置解析）（三者可能互转吗？比如 Effect Schema 能自动生成 JSON Schema 给 LLM 的 tool definition 用？答案是肯定的——`toJsonSchema()`）
- [ ] **Location 隔离**: 按工作目录创建独立的 Service Map。这意味着 `/project-a/` 和 `/project-b/` 下运行同一个 OpenCode 实例时，有各自独立的 Agent 列表、会话、配置缓存

### 依赖

- [ ] 无（本文是概览，不依赖其他文章）

---

## 编写指南

本文是全系列的"地图"，目标是让读者 10 分钟内建立全局认知。不要深入细节，每个点点到为止即可，但必须有代码引用（包路径、关键类型名）。

关键数据流图：
```
Config (opencode.jsonc) → Config.Service → AgentV2.layer → LocationServiceMap → 运行时状态
                                                        ↓
                                              PluginBoot → Plugin Loading → Hooks
                                                        ↓
                                              Session Run Loop → LLM → Tool → Result
```
