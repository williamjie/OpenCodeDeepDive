# 第三篇: Subagent 创建机制深度剖析

## 3.1 Subagent 概念

Subagent 是在主会话上下文中通过 `task` 工具派生的独立 AI 会话：

- **独立会话**: 每个 subagent 创建一个独立会话，拥有自己的上下文窗口
- **独立权限**: 权限受父 Agent 限制 + 自身配置共同决定
- **并行执行**: `run_in_background=true` 支持并行执行
- **会话延续**: `task_id="ses_xxx"` 保留上下文

## 3.2 Subagent vs Primary Agent

| 维度 | Primary Agent | Subagent |
|---|---|---|
| 可见性 | UI 可见，用户可切换 | 内部使用 |
| 调用方式 | 默认 Tab 切换 | `task()` 工具 |
| 生命周期 | 持久会话 | 任务级 |
| 权限继承 | 完整自身权限 | 父 deny + 自身权限 |
| 执行方式 | 串行 | 可串行/并行 |

## 3.3 内置 Subagent

**explore**: 只读搜索专用。仅允许 grep/glob/read/bash/webfetch/websearch，禁止任何写入。
**general**: 通用多步骤任务。全放开（除 todowrite）。

## 3.4 权限继承机制

核心文件：`packages/opencode/src/agent/subagent-permissions.ts`

```typescript
function deriveSubagentSessionPermission(input) {
  // 1. 继承父会话的 deny 规则和 external_directory 规则
  // 2. 子 Agent 没有显式配置 task → 禁止
  // 3. 子 Agent 没有显式配置 todowrite → 禁止
}
```

**核心原则**：父会话的 **deny 规则继承**，allow 规则**不继承**（Subagent 用自己的权限）。

## 3.5 Subagent 创建流程

```text
主会话 → task() 工具
  → 解析参数 (agent/category/prompt/background/task_id)
  → 解析 Agent 配置
  → deriveSubagentSessionPermission()
  → 创建/复用会话
  → 提交 prompt 并执行
  → 后台模式: 返回 bg_xxx，完成后 system-reminder 通知
  → 前台模式: 阻塞等待结果
  → 结果返回主会话
```

## 3.6 Category 到 Agent 映射

| category | 映射 Agent | 用途 |
|---|---|---|
| visual-engineering | 前端专用 | UI/UX/设计 |
| deep | general | 深度研究 |
| quick | 简化版 | 简单修改 |
| explore | explore | 代码搜索 |
| oracle | 只读咨询 | 架构/调试 |
| librarian | 文献搜索 | 外部库查询 |

---
