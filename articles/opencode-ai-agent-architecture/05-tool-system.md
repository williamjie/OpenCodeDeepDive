# 第五篇: 工具系统（Tool System）架构

## 5.1 V2 工具模型

核心文件：`packages/core/src/tool/tool.ts`

```typescript
const make: <Input, Output>(config: {
  description: string
  input: Input          // Effect Schema 定义输入
  output: Output        // Effect Schema 定义输出
  execute: (input, context: Tool.Context) => Effect.Effect<Output, ToolFailure>
  toModelOutput?: (input) => Tool.Content[]
}) => Definition<Input, Output>
```

**调用上下文**：
```typescript
interface Tool.Context {
  sessionID: Session.ID
  agent: Agent.ID
  assistantMessageID: Session.MessageID
  toolCallID: ToolCall.ID
}
```

## 5.2 内置工具清单

定义在 `packages/core/src/tool/builtins.ts`：

| 工具 | 功能 | 核心文件 |
|---|---|---|
| `read` | 读取文件，支持分页 | `read-filesystem.ts`, `read.ts` |
| `write` | 写入文件 | `write.ts` |
| `edit` | 编辑文件（精确字符串替换） | `edit.ts` |
| `grep` | 内容搜索 | `grep.ts` |
| `glob` | 文件模式匹配 | `glob.ts` |
| `bash` | 执行命令 | `bash.ts` |
| `websearch` | 网页搜索 | `websearch.ts` |
| `webfetch` | 网页获取 | `webfetch.ts` |
| `question` | 询问用户 | `question.ts` |
| `todowrite` | 创建管理 TODO | `todowrite.ts` |
| `skill` | 加载 skill | `skill.ts` |
| `apply_patch` | 应用补丁（新增/修改/删除） | `apply-patch.ts` |

## 5.3 注册与执行

**双层注册**：应用层 (ApplicationTools) + 位置层 (Tools.Service，优先级更高)

```text
Agent 请求 → LLM 解析 → 工具调用
  → ToolRegistry 查找 → 匹配注册名称
  → 权限检查 PermissionV2
  → 输入解码（Effect Schema）
  → 执行（Effect）
  → 输出编码 + Bound（大小限制）
  → 返回结果到 LLM
```

**Settlement 流程**：
```
1. 解析注册 → 找到有效的命名注册
2. 解码输入 → 如果无效，不执行工具
3. 调用工具 → 使用 runner 提供的上下文
4. 编码输出 → 如果无效，不返回成功
5. 投影输出 → toModelOutput 或结构化输出
6. 边界处理 → 过大内容截断到托管存储
```

## 5.4 输出边界处理

```typescript
// 通用输出边界
- 文本部分: 受限大小，超大的保存到托管存储
- 结构化元数据: 保留不变
- 原生媒体: 由生产者限制，保持原样
- 空内容: 度量结构化输出
```

## 5.5 工具注册示例

```typescript
// 注册一个自定义工具
tools.register({
  my_tool: Tool.make({
    description: "My custom tool",
    input: InputSchema,
    output: OutputSchema,
    execute: (input, context) =>
      Effect.gen(function* () {
        // 获取服务依赖
        const fs = yield* FileSystem.Service
        const permission = yield* PermissionV2.Service
        // 权限检查
        yield* permission.assert({ ... })
        // 执行
        return yield* fs.doSomething(input)
      }),
  }),
})
```

## 5.6 工具注册规则

- 同一名称最新注册生效
- 关闭注册只移除当前注册，暴露前一个
- Location 注册优先于应用注册
- 调用时捕获生效的注册，后续变更不影响运行中的调用
- 失效调用（注册被移除/替换后）被拒绝

---
