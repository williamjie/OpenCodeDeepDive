# 第八篇: 插件系统（Plugin System）扩展开发

> **用人话说**：插件就是"外挂"。不想改核心代码？装个插件就行。

---

## 插件架构

插件系统的核心接口：

| 方法 | 用人话说 |
|---|---|
| `add(id, effect)` | 安装插件 |
| `remove(id)` | 卸载插件 |

**用人话说**：就像手机装 App——装上就能用，卸载就没用了。

### 生命周期管理

使用 Effect Scope 管理插件生命周期：

- `Scope.fork` 创建子作用域
- `Scope.close` 清理资源

**用人话说**：就像给插件一个"沙箱"——插件在里面干活，卸载时整个沙箱清理干净。

---

## 插件钩子（Hooks）

插件可以挂载到系统的各个阶段：

| 钩子 | 用人话说 |
|---|---|
| `dispose` | 插件卸载时 |
| `event` | 事件发生时 |
| `config` | 配置加载时 |
| `tool` | 注册自定义工具 |
| `auth` | 自定义认证 |
| `provider` | Provider 扩展 |
| `chat.message` | 消息处理时 |
| `chat.params` | 修改 LLM 参数 |
| `chat.headers` | 修改请求头 |
| `permission.ask` | 权限询问时 |
| `tool.execute.before` | 工具调用前 |
| `tool.execute.after` | 工具调用后 |
| `shell.env` | 注入环境变量 |

**用人饭店**：就像给系统装了"钩子"——在特定阶段自动触发插件逻辑。

---

## V2 Effect Plugin 系统

V2 版本的插件系统基于 Effect 框架，提供更强大的能力：

| 模块 | 用人话说 |
|---|---|
| agent.ts | Agent 操作 |
| aisdk.ts | AI SDK 集成 |
| catalog.ts | Provider/Catalog |
| command.ts | 命令 |
| context.ts | 系统上下文 |
| event.ts | 事件 |
| filesystem.ts | 文件系统 |
| integration.ts | 集成 |
| location.ts | 位置 |
| npm.ts | NPM 包管理 |
| path.ts | 路径 |
| plugin.ts | 插件基础 |
| reference.ts | 引用 |
| registration.ts | 注册 |
| skill.ts | 技能 |

**用人话说**：就像给插件系统提供了一整套"工具箱"，插件可以用这些工具扩展系统能力。

---

## 插件热加载生命周期

```
Plugin.add(id, effect)
  → Scope.fork(scope) 创建子作用域
  → effect(host) 执行插件
  → 注册工具、钩子、Provider 等
  → Scope.close() 清理

Plugin.remove(id)
  → Scope.close(current) 关闭
  → 注册的变换自动回滚
```

**用人饭店**：就像热插拔 USB——插上就用，拔掉就自动清理，不用重启电脑。

### 重载检测

```
如果 loading.has(id) → 检测到循环依赖 → 报错
```

**用人饭店**：就像防止死循环——如果发现插件 A 依赖插件 B，插件 B 又依赖插件 A，就报错。

---

## 一句话总结

插件系统通过钩子机制扩展系统能力，支持热加载和自动清理，V2 版本基于 Effect 框架提供更强大的扩展能力。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-ai-agent-architecture/08-plugin-system.md)*
