# OpenCode 模型稳定性工程深度解析 — 系列文章

> 生成时间: 2026-06-23 09:16
> 文件名: 20260623-0916-opencode-model-instability-analysis.md
> 生成模型: deepseek-v4-flash-free

---

## 📋 概览

| 项目 | 信息 |
|---|---|
| **系列主题** | AI Agent 的 "80% 水下工程" — 模型不稳定性处理 |
| **核心问题** | LLM 输出不可靠、API 故障、JSON 异常、上下文溢出、权限误判 |
| **覆盖范围** | 重试策略、JSON Schema 验证、上下文窗口管理、权限守卫、工具防护、步骤限制、运行时恢复 |
| **代码包** | `packages/core`, `packages/opencode`, `packages/llm` |

> **背景**：在与不可预测的 LLM （幻觉、JSON 不稳定、API 超时、Token 耗尽等）打交道时，AI Agent 真正需要 80% 精力处理的是这些"水下工程"。OpenCode 在这方面构建了一套系统性的防御体系。

---

## 📑 文章索引

| 编号 | 文章标题 | 核心内容 |
|---|---|---|
| 一 | 重试与回退：应对 LLM API 不可靠 | 智能重试策略、错误分类、退避算法、Rate Limit 处理、免费额度 upsell |
| 二 | JSON Schema 验证：驯服模型输出 | Effect Schema 类型安全、结构化输出验证、解码/解析模式 |
| 三 | 上下文窗口管理：防止 Agent 记忆溢出 | Token 预估、溢出检测、自动压缩、LLM 辅助摘要生成 |
| 四 | 权限系统：防止模型误操作 | Ruleset 规则评估、Ask/Allow/Deny 模型、持久化权限记忆 |
| 五 | 工具系统防护：无效调用与陈旧调用 | Stale Tool Call 检测、工具定义验证、执行失败处理 |
| 六 | 运行时恢复：全方位容错机制 | Provider Error 处理、Tool Fiber 管理、中断恢复、Question Rejection |
| 七 | 设计哲学与工程实践 | 综合回顾：防御深度、Effect 函数式错误处理、可观测性 |

---
