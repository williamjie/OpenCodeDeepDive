# 第二篇: JSON Schema 验证 — 驯服模型输出

## 2.1 问题背景

LLM 的本质是概率生成器，其输出天然不稳定：

- JSON 格式错误（缺少括号、逗号错位）
- 字段缺失或类型错误
- 产生幻觉字段（hallucinated properties）
- 工具参数超出预期范围

## 2.2 Effect Schema：类型安全的验证体系

OpenCode 大量使用 Effect Schema（`@effect/schema`）来对 LLM 生成的输出进行运行时验证。Schema 模式将 TypeScript 的编译时类型系统延伸到运行时。

### 典型模式：Schema.Class + Schema.TaggedErrorClass

```typescript
// 定义结构化的错误类型
export class ModelNotSelectedError extends Schema.TaggedErrorClass<ModelNotSelectedError>()(
  "SessionRunnerModel.ModelNotSelectedError",
  { sessionID: SessionSchema.ID },
) {}

export class ModelUnavailableError extends Schema.TaggedErrorClass<ModelUnavailableError>()(
  "SessionRunnerModel.ModelUnavailableError",
  { providerID: ProviderV2.ID, modelID: ModelV2.ID },
) {}
```

**TaggedErrorClass** 确保错误可以被 Effect 的 `Cause` 系统精确匹配：
- 通过 `_tag` 字段（类名自动生成）实现模式匹配
- Effect 的 `Effect.catchTag()` 可以根据 `_tag` 精确捕获特定错误

### 数据模型 Schema 验证

```typescript
// 会话 ID 验证
export const ID = Schema.String.pipe(
  Schema.startsWith("ses"),        // 必须以 "ses" 开头
  Schema.brand("SessionV2.ID"),    // 品牌化类型
  withStatics((schema) => ({
    create: (id?: string) => schema.make(id ?? "ses_" + Identifier.ascending()),
  })),
)

// 权限请求 Schema
export const Request = Schema.Struct({
  id: ID,
  sessionID: SessionV2.ID,
  action: Schema.String,
  resources: Schema.Array(Schema.String),
  save: Schema.Array(Schema.String).pipe(Schema.optional),
  metadata: Schema.Record(Schema.String, Schema.Unknown).pipe(Schema.optional),
  source: Source.pipe(Schema.optional),
}).annotate({ identifier: "PermissionV2.Request" })
```

### Schema 解码的核心 API

Effect Schema 提供多个解码函数：

```typescript
// 严格解码：失败 → Error
Schema.decodeUnknown(schema)(data)
Schema.decodeUnknownEither(schema)(data)  // Either<ParseError, T>
Schema.decodeUnknownPromise(schema)(data)  // Promise<T>

// 异步解码（支持表单式验证）
Schema.decodeUnknownAsync(schema)(data)
Schema.decodeUnknownEffect(schema)(data)   // Effect<T, ParseError>

// 可选解码与容错
Schema.decodeUnknownOption(schema)(data)   // Option<T>
Schema.parse(schema)(data)                 // 同时解析和转换
Schema.asserts(schema)(data)               // 类型守卫
```

### 工具输入验证链

当 LLM 产生工具调用时，验证链如下：

```
LLM Output (text/JSON)
    │
    ▼
Schema.decodeUnknown (类型检查 + 结构验证)
    │
    ├── 成功 → 执行工具
    │
    └── 失败 → ParseError
          │
          ├── 缺失字段 → 返回错误给 LLM
          ├── 类型错误 → 返回错误给 LLM
          └── 多余字段 → 可配置忽略/报错
```

这种机制保证：即使 LLM 产生格式错误的工具调用，系统也不会崩溃，而是将错误信息反馈给 LLM 让其自我修正。

## 2.3 Permission Schema 验证体系

权限系统有自己的 Schema 层：

```typescript
export class RejectedError extends Schema.TaggedErrorClass<RejectedError>()(
  "PermissionV2.RejectedError", {}
) {}

export class CorrectedError extends Schema.TaggedErrorClass<CorrectedError>()(
  "PermissionV2.CorrectedError",
  { feedback: Schema.String },  // 包含用户反馈
) {}

export class DeniedError extends Schema.TaggedErrorClass<DeniedError>()(
  "PermissionV2.DeniedError",
  { rules: PermissionSchema.Ruleset },  // 显示被什么规则拒绝
) {}
```

**三层错误设计**：
- `DeniedError`：权限规则明确拒绝（确定性）
- `RejectedError`：用户拒绝（人为决策）
- `CorrectedError`：用户修改后拒绝（带有反馈信息，可用于 LLM 自我修正）

## 2.4 工具输出验证

工具执行结果也需要 Schema 验证：

```typescript
export const AskResult = Schema.Struct({
  id: ID,
  effect: PermissionSchema.Effect,  // "allow" | "deny" | "ask"
}).annotate({ identifier: "PermissionV2.AskResult" })
```

工具注册时，OpenCode 对工具名称进行验证：

```typescript
yield* Effect.forEach(entries, ([name]) => validateName(name), { discard: true })
```

---
