# 03 - Subagent 创建机制深度剖析

## 编写 TODO

### 需要阅读的源码

- [ ] 深入阅读 `packages/opencode/src/agent/subagent-permissions.ts`（27 行，短但精悍）
- [ ] 找到 `task` 工具的实现文件（可能在 `packages/opencode/src/tool/` 或 `packages/core/src/tool/` 下），理解 task 工具如何接收 agent / category / prompt / run_in_background / task_id 等参数
- [ ] 阅读 `packages/core/src/session/run-coordinator.ts`，理解 SessionRunCoordinator 的 resume / wake / interrupt 逻辑
- [ ] 在 `packages/opencode/` 或 `packages/core/` 中搜索 task 工具的分发路由逻辑（category → agent 的映射）

### 分析要点

- [ ] **subagent-permissions.ts 的继承算法**: `deriveSubagentSessionPermission()`
   ```typescript
   // 关键代码：只继承 parent 的 deny 和 external_directory 规则
   ...input.parentSessionPermission.filter(
     (rule) => rule.permission === "external_directory" || rule.action === "deny",
   )
   ```
   - 为什么只继承 deny？allow 为什么不继承？
   - 为什么 `external_directory` 规则总是继承？
- [ ] **默认禁止递归**: subagent 的 `task` 和 `todowrite` 工具被默认禁止——除非 subagent 自己的权限明确允许。这防止了"Agent 无限递归创建子 Agent"
- [ ] **Category → Agent 映射**: 深入追踪 task() 如何将 category（如 `visual-engineering`）解析为具体的 Agent 配置
- [ ] **Background vs Foreground**: `run_in_background=true` 返回 `bg_xxx` ID，完成后通过 `<system-reminder>` 通知；前台模式阻塞等待结果
- [ ] **会话延续**: `task_id="ses_xxx"` 保留上下文——底层是复用已有会话还是从历史重新构建？

### 深度洞察

- V1 权限继承用了极简规则——这是一个 "deny by default, explicit allow" 模型。父会话的 allow 对子会话无效，杜绝了权限过度扩散
- subagent 递归防护（默认禁止 task 工具）防止了 agent 创建的无限循环。如果需要 sub-subagent（如 explore 再创建 explore），必须显式在 agent 配置中放开
- `external_directory` 规则总是继承：这是文件系统访问控制的特殊性——如果父会话限制了 subagent 可以访问的目录，这个限制应该无条件传递到子会话

### 依赖

- [ ] 02 - Agent 系统
