# OpenCodeDeepDive

> OpenCode 源码深度解析 —— 从架构设计到工程实践
>
> OpenCode ([anomalyco/opencode](https://github.com/anomalyco/opencode)) 是一个开源 AI 编程助手，基于 TypeScript + Effect v4 构建，采用 Monorepo 架构。

---

## 阅读指南

本项目包含三个版本的文章系列，从不同层次解读 OpenCode 源码：

| 版本 | 目录 | 读者 | 特点 |
|---|---|---|---|
| **通俗版** 👈 起点 | `articles-popular/` | 技术负责人、架构师、泛技术人群 | 去代码化、类比丰富、聚焦设计决策 |
| **技术版** | `articles/` | 开发者、源码研究者 | 源码级解析、TypeScript 代码块、Effect 概念详解 |
| **分析笔记** 🔍 | `opencode-ai-agent-series/` | 希望深度学习的开发者 | 含 TODO 清单、分析要点、深度洞察 |

### 推荐阅读顺序

```
第一步：通俗版（建立全局认知，30 分钟）
    ↓
第二步：技术版（深入源码细节，按需阅读）
    ↓
第三步：分析笔记（跟 TODO 清单自己读源码，实践学习）
```

**一个主题三步走**：比如你对"权限系统"感兴趣 →
1. 读[通俗版](articles-popular/opencode-model-stability-engineering/04-permission-system.md) → 理解设计思路
2. 读[技术版](articles/opencode-model-stability-engineering/04-permission-system.md) → 看源码实现
3. 跟[分析笔记 TODO](opencode-ai-agent-series/TODO.md) → 自己读源码验证

---

## 系列一：AI Agent 架构设计

> 从零理解一个 AI Agent 系统是怎么搭起来的

| # | 通俗版 | 技术版 | 分析笔记 | 核心内容 |
|---|---|---|---|---|
| 一 | [架构总览](articles-popular/opencode-ai-agent-architecture/01-architecture-overview.md) | [技术版](articles/opencode-ai-agent-architecture/01-architecture-overview.md) | [笔记](opencode-ai-agent-series/01-architecture-overview.md) | 项目结构、技术选型、设计哲学 |
| 二 | [Agent 系统](articles-popular/opencode-ai-agent-architecture/02-agent-system.md) | [技术版](articles/opencode-ai-agent-architecture/02-agent-system.md) | [笔记](opencode-ai-agent-series/02-agent-system.md) | Agent 核心模型、内置角色、配置方式 |
| 三 | [Subagent 机制](articles-popular/opencode-ai-agent-architecture/03-subagent-mechanism.md) | [技术版](articles/opencode-ai-agent-architecture/03-subagent-mechanism.md) | [笔记](opencode-ai-agent-series/03-subagent-mechanism.md) | 主 Agent 派活给子 Agent |
| 四 | [多智能体协作](articles-popular/opencode-ai-agent-architecture/04-multi-agent-collaboration.md) | [技术版](articles/opencode-ai-agent-architecture/04-multi-agent-collaboration.md) | [笔记](opencode-ai-agent-series/04-multi-agent-collaboration.md) | 并行/串行协作模式 |
| 五 | [工具系统](articles-popular/opencode-ai-agent-architecture/05-tool-system.md) | [技术版](articles/opencode-ai-agent-architecture/05-tool-system.md) | [笔记](opencode-ai-agent-series/05-tool-system.md) | 工具注册与执行 |
| 六 | [会话系统](articles-popular/opencode-ai-agent-architecture/06-session-system.md) | [技术版](articles/opencode-ai-agent-architecture/06-session-system.md) | [笔记](opencode-ai-agent-series/06-session-system.md) | 会话管理与上下文压缩 |
| 七 | [技能与命令](articles-popular/opencode-ai-agent-architecture/07-skill-and-command.md) | [技术版](articles/opencode-ai-agent-architecture/07-skill-and-command.md) | [笔记](opencode-ai-agent-series/07-skill-and-command.md) | Skill 和 Command 系统 |
| 八 | [插件系统](articles-popular/opencode-ai-agent-architecture/08-plugin-system.md) | [技术版](articles/opencode-ai-agent-architecture/08-plugin-system.md) | [笔记](opencode-ai-agent-series/08-plugin-system.md) | 插件架构与热加载 |
| 九 | [LLM 运行时](articles-popular/opencode-ai-agent-architecture/09-llm-runtime.md) | [技术版](articles/opencode-ai-agent-architecture/09-llm-runtime.md) | [笔记](opencode-ai-agent-series/09-llm-runtime.md) | Provider 集成 |
| 十 | [最佳实践](articles-popular/opencode-ai-agent-architecture/10-best-practices.md) | [技术版](articles/opencode-ai-agent-architecture/10-best-practices.md) | [笔记](opencode-ai-agent-series/10-best-practices.md) | 总结与二次开发指南 |

---

## 系列二：模型稳定性工程

> AI Agent 80% 的工作是处理"不靠谱"的 LLM

| # | 通俗版 | 技术版 | 核心内容 |
|---|---|---|---|
| 一 | [重试与回退](articles-popular/opencode-model-stability-engineering/01-retry-and-fallback.md) | [技术版](articles/opencode-model-stability-engineering/01-retry-and-fallback.md) | API 故障处理 |
| 二 | [JSON 验证](articles-popular/opencode-model-stability-engineering/02-json-schema-validation.md) | [技术版](articles/opencode-model-stability-engineering/02-json-schema-validation.md) | LLM 输出格式校验 |
| 三 | [上下文管理](articles-popular/opencode-model-stability-engineering/03-context-window-management.md) | [技术版](articles/opencode-model-stability-engineering/03-context-window-management.md) | 记忆溢出处理 |
| 四 | [权限系统](articles-popular/opencode-model-stability-engineering/04-permission-system.md) | [技术版](articles/opencode-model-stability-engineering/04-permission-system.md) | 防止模型误操作 |
| 五 | [工具防护](articles-popular/opencode-model-stability-engineering/05-tool-system-protection.md) | [技术版](articles/opencode-model-stability-engineering/05-tool-system-protection.md) | 陈旧调用检测 |
| 六 | [运行时恢复](articles-popular/opencode-model-stability-engineering/06-runtime-recovery.md) | [技术版](articles/opencode-model-stability-engineering/06-runtime-recovery.md) | 全方位容错 |
| 七 | [设计哲学](articles-popular/opencode-model-stability-engineering/07-design-philosophy.md) | [技术版](articles/opencode-model-stability-engineering/07-design-philosophy.md) | 设计思路总结 |

---

## 其他资源

| 目录 | 内容 | 用途 |
|---|---|---|
| `backup/` | AI 生成的原始分析报告（单文件版） | 查阅原始生成记录 |
| `opencode-ai-agent-series/TODO.md` | 全局 TODO 清单 | 追踪分析进度，按图索骥读源码 |

---

## 快速开始

**如果你是技术负责人/架构师**：
1. 先看 [通俗版 README](articles-popular/README.md)
2. 按需阅读感兴趣的文章
3. 需要深入时再看对应的技术版

**如果你是开发者**：
1. 推荐从 [通俗版](articles-popular/opencode-ai-agent-architecture/01-architecture-overview.md) 入门
2. 读完通俗版对应章节后，看[技术版](articles/opencode-ai-agent-architecture/01-architecture-overview.md)
3. 跟 [分析笔记 TODO](opencode-ai-agent-series/TODO.md) 自己读源码

---

*基于 OpenCode 2026年6月 dev 分支分析*
