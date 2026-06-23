# 第八篇: 插件系统（Plugin System）扩展开发

## 8.1 插件架构

核心文件：`packages/core/src/plugin.ts`

```typescript
interface Interface {
  add(id: ID, effect: Plugin["effect"]): void   // 热加载
  remove(id: ID): void                           // 热卸载
}
```

使用 Effect Scope 生命周期管理：
- `Scope.fork` 创建子作用域
- `Scope.close` 清理资源

## 8.2 插件钩子（Hooks）

文件：`packages/plugin/src/index.ts`

```typescript
interface Hooks {
  dispose?: () => void
  event?: (input) => void
  config?: (input: Config) => void
  tool?: Record<string, ToolDefinition>         // 自定义工具
  auth?: AuthHook                                // 自定义认证
  provider?: ProviderHook                        // Provider 扩展

  // 会话相关
  "chat.message"?: (input, output) => void
  "chat.params"?: (input, output) => void        // 修改 LLM 参数
  "chat.headers"?: (input, output) => void       // 修改请求头

  // 权限
  "permission.ask"?: (input, output) => void

  // 工具拦截
  "tool.execute.before"?: (input, output) => void // 调用前修改参数
  "tool.execute.after"?: (input, output) => void  // 调用后修改结果

  // 环境
  "shell.env"?: (input, output) => void           // 注入环境变量

  // 实验性
  "experimental.chat.messages.transform"?: (input, output) => void
  "experimental.chat.system.transform"?: (input, output) => void
  "experimental.provider.small_model"?: (input, output) => void
  "experimental.session.compacting"?: (input, output) => void
}
```

## 8.3 V2 Effect Plugin 系统

`packages/plugin/src/v2/effect/` 定义了基于 Effect 的 V2 插件：

```
v2/effect/
├── agent.ts         # Agent 操作
├── aisdk.ts         # AI SDK 集成
├── catalog.ts       # Provider/Catalog
├── command.ts       # 命令
├── context.ts       # 系统上下文
├── event.ts         # 事件
├── filesystem.ts    # 文件系统
├── integration.ts   # 集成
├── location.ts      # 位置
├── npm.ts           # NPM 包管理
├── path.ts          # 路径
├── plugin.ts        # 插件基础
├── reference.ts     # 引用
├── registration.ts  # 注册
└── skill.ts         # 技能
```

## 8.4 插件热加载生命周期

```text
Plugin.add(id, effect)
  → Scope.fork(scope) 创建子作用域
  → effect(host) 执行插件
  → 注册工具、钩子、Provider 等
  → Scope.close() 清理

Plugin.remove(id)
  → Scope.close(current) 关闭
  → 注册的变换自动回滚
```

**重载检测**：
```typescript
const locks = KeyedMutex.makeUnsafe<ID>()  // 防止并发加载
const loading = new Set<ID>()               // 加载中检测
// 如果 loading.has(id) → 检测到循环依赖 → 报错
```

---
