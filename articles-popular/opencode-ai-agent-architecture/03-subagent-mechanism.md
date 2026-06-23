# 第三篇: Subagent 创建机制深度剖析

> **用人话说**：主 Agent 是"项目经理"，Subagent 是"干活的员工"。项目经理派活给员工，员工干完了汇报结果。

---

## Subagent 是什么？

Subagent 是在主会话上下文中通过 `task` 工具派生的**独立 AI 会话**。

用人话说：
- 主 Agent（比如 `build`）正在和你聊天
- 它觉得某个任务可以交给别人干
- 它通过 `task()` 派生一个 Subagent
- Subagent 独立干活，干完了把结果交回来

### Subagent 的特点

| 特点 | 用人话说 |
|---|---|
| 独立会话 | 每个 Subagent 有自己的上下文窗口，不和主 Agent 共享记忆 |
| 独立权限 | 权限受父 Agent 限制，但有自己的配置 |
| 并行执行 | `run_in_background=true` 可以同时派多个 Subagent |
| 会话延续 | `task_id="ses_xxx"` 可以继续之前的对话 |

---

## Subagent vs Primary Agent

| 维度 | Primary Agent（主角） | Subagent（配角） |
|---|---|---|
| 可见性 | UI 可见，用户能切换 | 内部使用，用户看不到 |
| 调用方式 | 默认 Tab 切换 | 通过 `task()` 工具 |
| 生命周期 | 持久会话，一直在 | 任务级，干完就结束 |
| 权限继承 | 完整自身权限 | 父 deny + 自身权限 |
| 执行方式 | 串行 | 可串行/并行 |

**用人话说**：主角是"常驻员工"，配角是"临时工"。临时工的权限受公司规定（父 deny）限制，但也有自己的岗位职责（自身权限）。

---

## 两个内置 Subagent

### explore — 只读搜索专用

- **权限**：只能 grep、glob、read、bash、webfetch、websearch
- **禁止**：任何写入操作
- **用途**：代码搜索、文件查找

### general — 通用多步骤任务

- **权限**：全放开（除了 todowrite）
- **用途**：复杂的研究和执行任务

---

## 权限继承机制

这是 Subagent 设计中最精妙的部分。

### 核心原则

> 父会话的 **deny 规则继承**，allow 规则**不继承**

用人话说：
- 父 Agent 说"不能删文件" → Subagent 也不能删
- 父 Agent 说"能读文件" → Subagent 不一定能读（要自己配）

### 继承流程

```
主会话 → task() 工具
  → 解析参数 (agent/category/prompt/background/task_id)
  → 解析 Agent 配置
  → deriveSubagentSessionPermission()
    1. 继承父会话的 deny 规则
    2. 子 Agent 没有显式配置 task → 禁止
    3. 子 Agent 没有显式配置 todowrite → 禁止
  → 创建/复用会话
  → 提交 prompt 并执行
  → 结果返回主会话
```

**用人话说**：Subagent 继承了父 Agent 的"禁止清单"，但自己的能力需要自己配置。

---

## Category 到 Agent 映射

当你用 `task(category="xxx")` 时，系统会自动映射到对应的 Agent：

| category | 映射 Agent | 用途 |
|---|---|---|
| visual-engineering | 前端专用 | UI/UX/设计 |
| deep | general | 深度研究 |
| quick | 简化版 | 简单修改 |
| explore | explore | 代码搜索 |
| oracle | 只读咨询 | 架构/调试 |
| librarian | 文献搜索 | 外部库查询 |

**用人话说**：你告诉系统"我需要前端专家"，系统自动派出对应的 Agent。

---

## 一句话总结

Subagent 是主 Agent 的"临时工"，通过 `task()` 派生，独立执行任务后汇报结果。权限继承父 Agent 的"禁止清单"，但有自己的能力配置。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-ai-agent-architecture/03-subagent-mechanism.md)*
