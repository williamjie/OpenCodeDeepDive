# OpenCode AI Agent 架构深度解析 — 通俗版

> 技术负责人的 AI Agent 架构参考

---

## 这个系列讲什么？

用 10 篇文章，从零拆解一个生产级 AI Agent 系统（OpenCode）的完整架构。不纠结代码细节，聚焦**设计决策**：为什么这样搭？有什么好处？踩过什么坑？

---

## 📑 文章索引

| 编号 | 文章 | 你能获得什么 |
|---|---|---|
| 一 | [架构总览](01-architecture-overview.md) | 全局视角：项目结构、技术选型、设计哲学 |
| 二 | [Agent 系统](02-agent-system.md) | 理解 Agent 的核心模型、内置角色、配置方式 |
| 三 | [Subagent 机制](03-subagent-mechanism.md) | 主 Agent 怎么"派活"给子 Agent |
| 四 | [多智能体协作](04-multi-agent-collaboration.md) | 多个 Agent 并行/串行协作的模式 |
| 五 | [工具系统](05-tool-system.md) | Agent 的"手和脚"——工具是怎么注册和执行的 |
| 六 | [会话系统](06-session-system.md) | Agent 的"记忆"——会话管理和上下文压缩 |
| 七 | [技能与命令](07-skill-and-command.md) | Agent 的"技能包"——Skill 和 Command 系统 |
| 八 | [插件系统](08-plugin-system.md) | 怎么给 Agent 装新能力——插件架构 |
| 九 | [LLM 运行时](09-llm-runtime.md) | AI 大脑怎么接进来——Provider 集成 |
| 十 | [最佳实践](10-best-practices.md) | 总结 + 二次开发指南 |

---

## 建议阅读顺序

```
第一篇（架构总览）→ 第二篇（Agent 系统）→ 第五篇（工具系统）→ 第六篇（会话系统）
```

这 4 篇覆盖了核心骨架。其余篇章可以按需跳读。

---

*通俗版，非技术细节版。想看源码级解析请看 `../articles/`*
