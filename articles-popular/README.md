# OpenCode 深度解析 — 通俗版

> 技术负责人的 AI Agent 架构参考手册

---

## 这是什么？

这是 OpenCode 源码解析的**通俗版**系列文章。我们用更直白的语言、更少的代码、更多的类比，帮你快速理解一个生产级 AI Agent 系统的核心设计。

**适合谁看？**
- 技术负责人：评估 AI Agent 架构选型
- 架构师：了解多 Agent 协作、会话管理、插件系统的设计模式
- 泛技术人群：对 AI Agent 底层实现好奇

**不适合谁？**
- 想看源码实现细节的开发者 → 请看 `articles/` 目录的技术版

---

## 📑 系列一：AI Agent 架构设计

> 从零理解一个 AI Agent 系统是怎么搭起来的

| 编号 | 文章 | 一句话摘要 |
|---|---|---|
| 一 | [架构总览](opencode-ai-agent-architecture/01-architecture-overview.md) | OpenCode 长什么样，用了什么技术 |
| 二 | [Agent 系统](opencode-ai-agent-architecture/02-agent-system.md) | Agent 是什么，有哪些内置角色 |
| 三 | [Subagent 机制](opencode-ai-agent-architecture/03-subagent-mechanism.md) | 主 Agent 怎么派活给子 Agent |
| 四 | [多智能体协作](opencode-ai-agent-architecture/04-multi-agent-collaboration.md) | 多个 Agent 怎么并行干活 |
| 五 | [工具系统](opencode-ai-agent-architecture/05-tool-system.md) | Agent 的手和脚——工具 |
| 六 | [会话系统](opencode-ai-agent-architecture/06-session-system.md) | Agent 的记忆——会话管理 |
| 七 | [技能与命令](opencode-ai-agent-architecture/07-skill-and-command.md) | Agent 的技能包和快捷指令 |
| 八 | [插件系统](opencode-ai-agent-architecture/08-plugin-system.md) | 怎么给 Agent 装新能力 |
| 九 | [LLM 运行时](opencode-ai-agent-architecture/09-llm-runtime.md) | AI 大脑怎么接进来 |
| 十 | [最佳实践](opencode-ai-agent-architecture/10-best-practices.md) | 总结 + 二次开发指南 |

---

## 📑 系列二：模型稳定性工程

> AI Agent 80% 的工作是处理"不靠谱"的 LLM

| 编号 | 文章 | 一句话摘要 |
|---|---|---|
| 一 | [重试与回退](opencode-model-stability-engineering/01-retry-and-fallback.md) | API 挂了怎么办 |
| 二 | [JSON 验证](opencode-model-stability-engineering/02-json-schema-validation.md) | LLM 输出格式不对怎么办 |
| 三 | [上下文管理](opencode-model-stability-engineering/03-context-window-management.md) | 记忆太多装不下怎么办 |
| 四 | [权限系统](opencode-model-stability-engineering/04-permission-system.md) | 防止 Agent 乱来 |
| 五 | [工具防护](opencode-model-stability-engineering/05-tool-system-protection.md) | 防止 Agent 调错工具 |
| 六 | [运行时恢复](opencode-model-stability-engineering/06-runtime-recovery.md) | 各种异常都能兜住 |
| 七 | [设计哲学](opencode-model-stability-engineering/07-design-philosophy.md) | 背后的设计思路 |

---

## 与技术版的区别

| 维度 | 技术版 (`articles/`) | 通俗版 (`articles-popular/`) |
|---|---|---|
| 代码量 | 大量 TypeScript 代码块 | 几乎没有代码 |
| 术语 | 直接用 Effect/Layer/Context.Service | 用类比解释，保留英文术语 |
| 标题风格 | `## 1.1 问题背景` | `## 为什么 LLM API 让人头疼` |
| 读者 | 开发者、源码研究者 | 技术负责人、架构师 |
| 深度 | 源码级 | 设计决策级 |

---

*基于 OpenCode 2026年6月 dev 分支分析*
