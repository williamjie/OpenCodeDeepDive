# 第六篇: 会话系统（Session System）设计

> **用人话说**：Agent 的"记忆"。会话系统管理 Agent 的对话历史、上下文和状态。

---

## 会话是什么？

会话是 Agent 与用户交互的完整上下文，包括：

| 部分 | 用人话说 |
|---|---|
| `id` | 会话的唯一标识 |
| `metadata` | 标题、使用的 Agent、模型、花费、token 用量 |
| `messages` | 所有消息：用户输入、Agent 回复、工具调用结果 |
| `events` | 事件流：发生了什么事 |
| `context_epoch` | 系统上下文快照 |

**用人话说**：会话就像一个"聊天记录本"，记录了所有对话内容和发生的事情。

---

## 会话 API

核心操作：

| 操作 | 用人话说 |
|---|---|
| create | 创建新会话 |
| get | 获取会话 |
| list | 列出所有会话 |
| prompt | 发送消息 |
| messages | 获取消息列表 |
| context | 获取上下文 |
| resume | 恢复会话 |
| interrupt | 中断会话 |
| switchAgent | 切换 Agent |
| switchModel | 切换模型 |
| compact | 压缩上下文 |

---

## Context Epoch：上下文纪元

这是 V2 的核心创新之一。

### 什么是 Context Epoch？

每次 LLM 请求时，系统上下文（System Context）会被记录为一个 Context Epoch。

用人话说：就像给"当前状态"拍一张快照。

### 包含什么？

```
Context Epoch 包含：
├── 环境事实（环境变量、日期）
├── 项目指令（AGENTS.md）
├── 配置指令（instructions）
├── Agent 技能提示（skills）
└── 外部引用（references）
```

### 怎么用？

```
第一次请求 → 初始化 Context Epoch
后续请求 → 对比变化
  ├── 无变化 → 复用基线（省 token）
  ├── 有变化 → 生成系统消息 → 更新快照
  └── 自动压缩 → 重建完整基线
```

**用人话说**：就像比较两个版本的文档——如果没改过，就用旧的；如果改了，就生成新的。

---

## 自动压缩（Compaction）

当上下文接近模型限制时自动触发。

### 触发时机

1. **预检**：每次 LLM 请求前，估算 token 用量，如果快满了就先压缩
2. **后置**：LLM 返回"上下文溢出"错误时，被动压缩

### 压缩策略

```
对话历史 → 分割为"历史部分"和"最新部分"
历史部分 → LLM 生成结构化摘要
最新部分 → 保持原始内容
下次压缩 → 合并历史摘要 + 最新历史 → 生成新摘要
```

**用人话说**：就像整理笔记——把旧的笔记总结成摘要，保留最近的详细内容。下次整理时，把摘要和新笔记一起总结。

### 结构化摘要模板

压缩时，LLM 被要求生成包含 8 个固定段落的摘要：

1. Goal（目标）
2. Constraints & Preferences（约束和偏好）
3. Progress（进度）
4. Key Decisions（关键决策）
5. Next Steps（下一步）
6. Critical Context（关键上下文）
7. Relevant Files（相关文件）

**用人话说**：就像项目周报——必须包含目标、进度、问题、下一步等固定内容。

### 安全设计

- **禁止提及压缩过程**：避免让 LLM 在回复中暴露"系统提示"（越狱保护）
- **保留精确引用**：文件路径、错误信息、命令等关键信息必须原样保留

---

## 递送模式

| 模式 | 行为 | 用人话说 |
|---|---|---|
| steer | 推进到下一个 provider turn 边界 | 直接干活 |
| queue | 进入 FIFO 队列，当前 drain 结束后自动提升 | 排队等 |

**用人饭店**：steer 是"插队"，queue 是"排队"。

---

## 事件驱动

Session 采用事件溯源，所有状态变更记录为持久化事件。

| 事件 | 触发点 |
|---|---|
| Compaction.Started | 压缩开始 |
| Compaction.Ended | 压缩完成 |
| PromptAdmitted | 输入被接收 |
| Prompted | 输入已投递 |
| AgentSwitched | Agent 切换 |
| ModelSwitched | 模型切换 |

**用人话说**：就像银行账本，每一笔交易都记着。想知道某个时刻的状态？重放事件就知道了。

---

## 一句话总结

会话系统是 Agent 的"记忆"，通过 Context Epoch 智能管理上下文，支持自动压缩防止记忆溢出，事件溯源确保可追溯。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-ai-agent-architecture/06-session-system.md)*
