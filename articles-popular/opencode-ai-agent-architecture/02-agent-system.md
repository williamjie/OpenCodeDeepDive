# 第二篇: AI Agent 系统设计与实现

> **用人话说**：Agent 就是 AI 的"角色"。不同的 Agent 有不同的性格、能力和权限，就像一个团队里有程序员、产品经理、测试工程师。

---

## Agent 是什么？

在 OpenCode 里，Agent 不是一个抽象概念，而是一个**具体的配置对象**。每个 Agent 有这些属性：

| 属性 | 用人话说 |
|---|---|
| `id` | 名字，比如 `build`、`plan` |
| `model` | 用哪个 AI 模型，比如 Claude、GPT |
| `system` | 系统提示词——告诉 AI "你是谁、该干什么" |
| `description` | 一句话描述 |
| `mode` | 角色类型：主角还是配角 |
| `hidden` | 是否对用户隐藏 |
| `steps` | 最多能干几步（防止死循环） |
| `permissions` | 能用什么工具、不能用什么工具 |

### Agent 的三种模式

| 模式 | 用人话说 | 例子 |
|---|---|---|
| **primary** | 主角，用户能看到、能切换 | `build`（开发）、`plan`（规划） |
| **subagent** | 配角，内部干活用 | `explore`（搜索）、`general`（通用） |
| **all** | 既能当主角也能当配角 | 用户自定义的 Agent |

---

## 七个内置 Agent

OpenCode 内置了 7 个 Agent，每个都有明确的分工：

### build — 默认开发 Agent

- **角色**：日常编码开发
- **权限**：全放开，但有些操作需要用户确认
- **特点**：这是你最常用的 Agent

### plan — 计划/只读 Agent

- **角色**：代码审查与规划
- **权限**：**禁止所有编辑工具**，只能读不能写
- **特点**：让它看代码可以，让它改代码不行

### general — 通用子 Agent

- **角色**：多步骤任务执行
- **权限**：全放开（除了 todowrite）
- **特点**：适合复杂的研究和执行任务

### explore — 代码探索专用

- **角色**：代码搜索
- **权限**：**只读**——只能 grep、glob、read、bash、websearch
- **特点**：让它搜代码可以，让它改代码不行

### compaction / title / summary — 隐藏 Agent

这三个 Agent 对用户隐藏，专门干后台活：

| Agent | 干什么 | 用人话说 |
|---|---|---|
| compaction | 上下文压缩 | 记忆太多时自动"压缩"旧对话 |
| title | 标题生成 | 自动给会话起标题 |
| summary | 会话总结 | 自动总结会话内容 |

---

## 用户自定义 Agent

通过 `opencode.jsonc` 配置文件，你可以创建自己的 Agent：

```
{
  "agents": {
    "reviewer": {
      "model": "openai/gpt-5",
      "description": "Review changes",
      "system": "You are a code reviewer...",
      "mode": "subagent",
      "color": "warning",
      "steps": 12
    }
  }
}
```

### 配置覆盖规则

1. `disable: true` → 删除该 Agent
2. 同名已存在 → 合并覆盖（model、prompt 等替换，permission 合并）
3. 同名不存在 → 创建新的自定义 Agent

**用人话说**：你可以"魔改"内置 Agent，也可以创建全新的 Agent。

---

## Agent 的生命周期

```
配置文件 → Config.Service → Agent 状态管理 → 运行时可查询
```

选择逻辑（当你指定一个 Agent 时）：

```
1. 按 ID 查找 → 找到了就用
2. 配置里的默认值 → 用配置的
3. build → 用默认的 build
4. 第一个可见的 primary → 兜底
```

**用人话说**：先找你指定的，找不到就用默认的，再找不到就用最基础的。

---

## 一句话总结

Agent 是 OpenCode 的核心概念，每个 Agent 是一个有明确角色、权限和能力的 AI 实例。内置 7 个 Agent 覆盖常见场景，用户也可以通过配置文件自定义。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-ai-agent-architecture/02-agent-system.md)*
