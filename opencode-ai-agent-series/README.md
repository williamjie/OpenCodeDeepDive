# OpenCode AI Agent 架构深度解析 — 10 篇系列文章

> 目标：通过对 OpenCode 源码的深度阅读，揭示生产级 AI Agent 系统的架构设计、工程取舍与实现细节。
> 
> 代码路径: `packages/core/`, `packages/opencode/`, `packages/plugin/`, `packages/llm/`, `specs/v2/`
> 
> 每篇文章包含 TODO 清单，标记需要深入分析的源码文件与关键思考点。

---

## 系列结构

| # | 标题 | 核心源码目录 | 篇幅 |
|---|------|-------------|------|
| 01 | OpenCode 架构总览与核心技术栈 | 全项目 | 概览 |
| 02 | AI Agent 系统设计与实现 | `packages/core/src/agent.ts`, `packages/opencode/src/agent/` | 中 |
| 03 | Subagent 创建机制深度剖析 | `subagent-permissions.ts`, `task()` 工具 | 中 |
| 04 | 多智能体协作机制 | `run-coordinator.ts`, `background-job.ts`, `runner/llm.ts` | 中 |
| 05 | 工具系统架构 | `tool/tool.ts`, `tool/registry.ts`, `tool/builtins.ts` | 大 |
| 06 | 会话系统设计 | `session/schema.ts`, `store.ts`, `message.ts`, `sql.ts` | 大 |
| 07 | 技能系统与命令系统 | `skill.ts`, `.opencode/command/` | 中 |
| 08 | 插件系统扩展开发 | `plugin/src/index.ts`, `plugin/src/v2/effect/` | 大 |
| 09 | LLM 运行时与 Provider 集成 | `runner/llm.ts`, `runner/model.ts`, `system-context/` | 大 |
| 10 | 技术亮点与最佳实践 | 全项目回顾 | 中 |
