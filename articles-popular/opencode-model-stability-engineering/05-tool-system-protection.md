# 第五篇: 工具系统防护 — 无效调用与陈旧调用

> **用人话说**：Agent 可能会调用"过期"的工具，或者调用"不存在"的工具。工具系统防护就是防止这种情况。

---

## 为什么需要工具防护？

LLM 生成的工具调用可能：

| 问题 | 用人话说 |
|---|---|
| Stale（陈旧） | LLM 在切换上下文后，仍然调用上一次的工具 |
| Unknown（未知） | LLM 幻觉出一个不存在的工具 |
| Permission Denied（权限不足） | LLM 试图调用被禁止的工具 |

---

## Stale Tool Call 检测

在 `ToolRegistry.materialize()` 中，每个工具注册时都有一个唯一的身份令牌。

### 工作原理

```
1. 每次 materialize() 创建一个新的空对象 {} 作为身份标识
2. 工具定义（definitions）通过闭包捕获这个身份标识
3. 工具执行时，LLM 传回的 advertised 与 registry 中的 registration.identity 比较
4. 如果不一致 → Stale Tool Call → 返回错误
```

**用人饭店**：就像给每个工具发一张"身份证"——每次注册都发新证，执行时要核对身份证。如果身份证过期了，就拒绝执行。

### 流程图

```
materialize() ──→ 创建 identity = {}
                    │
                    ▼
               注册到 definitions（捕获 identity）
                    │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    LLM 调用    重新 materialize   LLM 调用
    identity=①    → identity=②    identity=①
        │                         │
        ▼                         ▼
    匹配 ✓ 执行             不匹配 ✓ 返回 Stale Error
```

---

## 工具注册生命周期

工具注册使用 Effect 的 `Scope` 和 `addFinalizer` 实现自动清理。

### 设计亮点

| 特性 | 用人话说 |
|---|---|
| 使用 `Effect.Scope` 实现自动注册/注销 | 装上就用，卸载自动清理 |
| 多个注册者同名工具时使用栈结构 | 后注册的优先 |
| 在 Scope 清理时自动恢复前一个注册 | 卸载时恢复之前的版本 |

**用人饭店**：就像插拔 USB 设备——插上就用，拔掉就自动清理，不影响其他设备。

---

## 工具执行失败的类型安全处理

当工具执行失败时，通过 `Effect.catchTag` 精确匹配 LLM 错误：

```
工具执行失败 → catchTag("LLM.ToolFailure") → 返回错误给 LLM
```

**用人饭店**：就像错误分类——不同类型的错误有不同的处理方式，不会混淆。

---

## 工具权限过滤

在 `materialize()` 阶段，会对工具进行权限过滤：

```
工具列表 → 按权限过滤 → 可用工具列表
```

`whollyDisabled()` 检查工具的 action 是否被完全禁止：

```
如果规则匹配 action 且 resource="*" 且 effect="deny" → 完全禁止
```

**用人饭店**：就像员工权限——不是所有人都能用所有工具，有些工具只有特定角色才能用。

---

## 一句话总结

工具系统防护通过身份令牌检测陈旧调用、Scope 管理自动清理、权限过滤防止越权，确保 Agent 只能调用有效且允许的工具。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-model-stability-engineering/05-tool-system-protection.md)*
