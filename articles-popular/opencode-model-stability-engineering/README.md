# OpenCode 模型稳定性工程深度解析 — 通俗版

> AI Agent 80% 的工作是处理"不靠谱"的 LLM

---

## 这个系列讲什么？

LLM（大语言模型）是 AI Agent 的大脑，但这个大脑很"不靠谱"：
- 输出格式不稳定（有时 JSON 格式错误）
- API 经常超时或报错
- 上下文窗口有限（记忆会溢出）
- 可能会"幻觉"出不存在的工具

这个系列用 7 篇文章，拆解 OpenCode 如何系统性地处理这些问题。

---

## 📑 文章索引

| 编号 | 文章 | 你能获得什么 |
|---|---|---|
| 一 | [重试与回退](01-retry-and-fallback.md) | API 挂了怎么办 |
| 二 | [JSON 验证](02-json-schema-validation.md) | LLM 输出格式不对怎么办 |
| 三 | [上下文管理](03-context-window-management.md) | 记忆太多装不下怎么办 |
| 四 | [权限系统](04-permission-system.md) | 防止 Agent 乱来 |
| 五 | [工具防护](05-tool-system-protection.md) | 防止 Agent 调错工具 |
| 六 | [运行时恢复](06-runtime-recovery.md) | 各种异常都能兜住 |
| 七 | [设计哲学](07-design-philosophy.md) | 背后的设计思路 |

---

## 建议阅读顺序

```
第一篇（重试）→ 第二篇（JSON 验证）→ 第三篇（上下文管理）→ 第七篇（设计哲学）
```

这 4 篇覆盖了核心防御机制。其余篇章可以按需跳读。

---

*通俗版，非技术细节版。想看源码级解析请看 `../articles/`*
