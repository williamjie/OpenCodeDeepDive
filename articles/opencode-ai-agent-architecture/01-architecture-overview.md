# 第一篇: OpenCode 架构总览与核心技术栈

## 1.1 项目全景

OpenCode 是一个完整的开源 AI 编程助手，对标 Claude Code，提供多平台接入能力：

- **TUI 终端界面** (`packages/tui`) — 命令行交互界面，类似 Claude Code 的终端体验
- **Desktop 桌面应用** (`packages/desktop`) — Electron 封装的原生桌面应用
- **Web 控制台** (`packages/console/app`) — 浏览器端控制台
- **Web App** (`packages/web`, `packages/app`) — SolidStart 构建的 Web 应用
- **CLI 命令行** (`packages/cli`) — 命令行入口
- **VSCode SDK 集成** (`sdks/vscode`) — VSCode 扩展
- **JavaScript SDK** (`packages/sdk`, `packages/sdk/js`) — 编程 SDK

## 1.2 Monorepo 包结构

OpenCode 采用 Turborepo + Bun workspaces 管理 25 个包：

### 核心层 (Core)

| 包路径 | 职责 | 依赖等级 |
|---|---|---|
| `packages/core` | **领域模型、Schema、事件系统、会话管理(V2)、工具系统、插件钩子、配置系统** | 核心依赖 |
| `packages/opencode` | **应用层编排、Agent 管理、Provider 适配、会话处理、CLI/Server 入口** | 核心依赖 |
| `packages/llm` | 原生 LLM 运行时 (`@opencode-ai/llm`)，LLMClient / RequestExecutor | 核心依赖 |

### 插件与扩展

| 包路径 | 职责 |
|---|---|
| `packages/plugin` | 插件 SDK，供外部开发者开发插件 |
| `packages/sdk` | SDK 和 API 客户端生成 |
| `packages/script` | 脚本工具 |

### UI 层

| 包路径 | 职责 |
|---|---|
| `packages/tui` | 终端 UI 界面 (SolidJS + opentui) |
| `packages/ui` | 共享 UI 组件库 |
| `packages/web` | Web 应用 (lander/登录页) |
| `packages/app` | Web 主应用 (SolidStart) |
| `packages/console/app` | Console 管理端 |
| `packages/storybook` | 组件文档 (Storybook) |

### 桌面端

| 包路径 | 职责 |
|---|---|
| `packages/desktop` | 桌面应用 (Electron) |

### 基础设施

| 包路径 | 职责 |
|---|---|
| `packages/effect-drizzle-sqlite` | Effect + Drizzle ORM + SQLite 集成 |
| `packages/effect-sqlite-node` | Effect SQLite Node 适配 |
| `packages/identity` | 身份认证服务 |
| `packages/server` | 后端服务器 |
| `packages/stats` | 统计面板 (SST) |
| `packages/http-recorder` | HTTP 录制调试工具 |
| `packages/function` | 云函数 |
| `packages/enterprise` | 企业版功能 |

### 文档与社区

| 包路径 | 职责 |
|---|---|
| `packages/docs` | 项目文档 |
| `packages/containers` | Docker 容器化 |
| `packages/slack` | Slack 集成 |

## 1.3 核心技术栈详解

### TypeScript + Bun

- 运行时: **Bun v1.3.14**（替代 Node.js）
- 语言: 全 TypeScript，开启 strict 模式
- 构建: Turborepo 管理 pipeline
- 测试: `bun test`
- 类型检查: `bun typecheck`（而非 tsc）
- Linter: `oxlint`（替代 ESLint）
- 格式化: Prettier

### Effect v4 函数式编程框架

这是 OpenCode 最核心、最具特色的技术选型。Effect v4 提供了远超 async/await 的能力：

**核心概念映射：**

| Effect 概念 | 类似的传统概念 | 在 OpenCode 中的应用 |
|---|---|---|
| `Effect.gen(function* () { ... })` | async/await + try/catch | 所有异步逻辑编排 |
| `Context.Service` | 依赖注入容器 | 服务定义与获取 |
| `Layer.effect()` | 依赖注册 | 服务实现注册 |
| `Schema.Class()` | Zod/Zod 对象 | 类型定义 + 运行时验证 |
| `Scope` | try/resource | 资源生命周期管理 |
| `SynchronizedRef` | Mutex | 并发状态管理 |
| `Schema.TaggedErrorClass` | 类型错误 | 类型化错误处理 |

**典型服务模式：**
```typescript
// 1. 定义接口
export interface Interface {
  readonly get: (id: ID) => Effect.Effect<Info | undefined>
  readonly list: () => Effect.Effect<Info[]>
}

// 2. 注册 Service
export class Service extends Context.Service<Service, Interface>()("@opencode/v2/Agent") {}

// 3. 实现 Layer
export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const config = yield* Config.Service  // 自动注入
    // 实现逻辑...
    return Service.of({ get, list })
  }),
)

// 4. 使用
const agent = yield* AgentV2.Service
const info = yield* agent.get("build")
```

### AI SDK

- **默认运行时**: Vercel AI SDK (`ai` npm package) 的 `streamText` / `generateObject`
- **原生运行时**: `@opencode-ai/llm`（opt-in，通过 `OPENCODE_EXPERIMENTAL_NATIVE_LLM=true` 启用）
- **支持的 Provider**: OpenAI、Anthropic、Google、xAI、OpenRouter 等
- **结构化输出**: 使用 AI SDK 的 `generateObject` 配合 Effect Schema

### Drizzle ORM + SQLite

- **ORM**: Drizzle ORM (v1.0.0-rc.2)
- **数据库**: SQLite (通过 Effect 管理事务)
- **Schema**: 定义在 `packages/core/src/session/sql.ts`

## 1.4 V2 架构设计哲学

从 `specs/v2/instructions.md` 提炼的设计原则：

### 原则一：Core 薄层

> "Move behavior out of large application services and into plugins. Core services should become small, typed containers that own state, expose simple operations, and trigger hooks where policy or integration-specific logic belongs."

Core 包只应包含：
- 领域 Schema 和类型错误
- 状态容器
- 事件定义
- 插件钩子合约

### 原则二：插件化

> "Plugins implement provider-specific, config-specific, auth-specific, model-discovery, and generation behavior."

业务逻辑通过插件实现，核心不包含应用策略。

### 原则三：可重放变换

> "Services are hot-reloadable by design: updates are granular, observable, and do not require tearing down the whole process."

配置变更和插件加载通过可重放的变换（replayable transform）实现。

### 原则四：事件溯源

会话使用事件溯源模式，所有状态变更记录为持久化事件流。

### 原则五：Location 作用域

服务实例按工作目录（Location）缓存不同目录有独立的服务实例和状态。

## 1.5 配置发现机制

OpenCode 支持多级配置，按优先级从低到高：

1. **全局配置**: `~/.config/opencode/opencode.jsonc`
2. **项目配置文件**: 从项目目录往上查找 `opencode.json` / `opencode.jsonc`
3. **`.opencode` 目录**: `{project}/.opencode/opencode.jsonc`（最高优先级）

**规则**：越具体的配置优先级越高，权限规则使用相反的顺序（全局规则可以覆盖项目规则）。

---
