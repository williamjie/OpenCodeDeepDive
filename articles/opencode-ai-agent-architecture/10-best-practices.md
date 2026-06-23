# 第十篇: 技术亮点与二次开发指南

## 10.1 架构亮点总结

1. **Event Sourcing + CQRS**: 会话完整事件溯源，支持回放和审计
2. **Effect 函数式编程**: 全栈 Effect v4，统一异步、DI、错误处理、并发
3. **Location 隔离**: 按工作目录隔离服务实例
4. **插件热加载**: 动态加载/卸载，无需重启
5. **Context Epoch**: 智能上下文管理，自动压缩
6. **Eager Tool Execution**: 主动工具执行，减少延迟
7. **双层 LLM 运行时**: AI SDK 默认 + 原生运行时 opt-in
8. **多层权限系统**: 默认权限 + Agent 权限 + 父会话继承 + 用户配置

## 10.2 值得关注的业务场景

1. **企业 AI 编程助手**: 完整的参考实现
2. **多 Agent 工作流引擎**: Orchestrator-Worker + 并行/串行委派
3. **可扩展工具平台**: 插件 SDK + 工具注册机制
4. **会话回溯与调试**: 事件溯源支持完整回放
5. **技能生态**: Skill 作为 AI 知识分发单元
6. **AI Agent 生成**: 让 LLM 自己编写 Agent 配置

## 10.3 二次开发关键文件

| 关注点 | 路径 |
|---|---|
| Agent 定义与配置 | `packages/opencode/src/agent/agent.ts` |
| Agent 核心模型 | `packages/core/src/agent.ts` |
| Subagent 权限 | `packages/opencode/src/agent/subagent-permissions.ts` |
| Agent 配置 Schema | `packages/core/src/config/agent.ts` |
| 工具系统 | `packages/core/src/tool/` (20 文件) |
| 会话管理 | `packages/core/src/session.ts` |
| 会话执行器 | `packages/core/src/session/runner/` |
| 插件 SDK | `packages/plugin/src/` |
| V2 插件 Effect | `packages/plugin/src/v2/effect/` (18 文件) |
| 技能系统 | `packages/core/src/skill.ts` |
| 背景任务 | `packages/core/src/background-job.ts` |
| 配置系统 | `packages/core/src/config.ts` |
| LLM 运行时 | `packages/opencode/src/session/llm/` (4 文件) |
| 原生 LLM | `packages/llm/` |
| V2 架构规格 | `specs/v2/` (10 文件) |
| 事件系统 | `packages/core/src/event.ts` |
| Provider 目录 | `packages/core/src/catalog.ts` |
| 插件钩子宿主 | `packages/core/src/plugin/host.ts` |

## 10.4 核心流程速查

### Agent 创建流程
```text
config → Config.Service → AgentV2.layer → 状态管理 → 查询
```

### Subagent 生成流程
```text
task() → 解析 Agent → 权限计算 → 创建会话 → 执行 → 返回
```

### 模型选择流程
```text
Session → Agent → Agent.model → Provider → Auth → API
```

### 插件加载流程
```text
config → PluginBoot → 安装 → 激活 → transform → 生效
```

## 10.5 系列学习路线

- **第一篇: 架构总览** → 建立全局认知
- **第二篇: Agent 系统** → 理解核心概念
- **第三篇: Subagent** → 掌握多 Agent 基础
- **第四篇: 多 Agent 协作** → 理解协作模式
- **第五篇: 工具系统** → 理解 Agent 能力边界
- **第六篇: 会话系统** → 深入运行机制
- **第七篇: 技能与命令** → 扩展 Agent 能力
- **第八篇: 插件系统** → 掌握扩展开发
- **第九篇: LLM 运行时** → 底层 AI 集成
- **第十篇: 最佳实践** → 总结与二次开发指南

## 10.6 关注的技术细节

1. **Effect 的优势与学习曲线**: 类型安全 + 依赖注入，但函数式编程有门槛
2. **双重 Schema 系统**: Effect Schema + Drizzle Schema + Config Schema
3. **权限系统的多层叠加**: 默认 → Agent → 父会话 → 用户
4. **V1 到 V2 的迁移**: 并行存在 `packages/core/src/v1/` 目录
5. **Durable vs In-memory**: 会话持久化 vs BackgroundJob 内存管理
6. **OpenTelemetry 集成**: 实验性功能，需要在配置中启用

---

## 📚 参考资源

- **代码仓库**: https://github.com/anomalyco/opencode
- **官方文档**: https://opencode.ai/docs
- **Effect v4**: https://effect.website
- **AI SDK**: https://sdk.vercel.ai
- **V2 设计文档**: `specs/v2/` 目录 (session.md, tools.md, instructions.md, config.md 等)
- **Drizzle ORM**: https://orm.drizzle.team
- **分析基准**: 2026年6月 dev 分支

---

*本系列文章通过对 OpenCode 源码的深度解析产出，重点聚焦 AI Agent 搭建、Subagent 创建、多智能体协作、工具系统、会话管理、技能/插件系统等核心主题。*
