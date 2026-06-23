# 第二篇: JSON Schema 验证 — 驯服模型输出

> **用人话说**：LLM 输出的 JSON 经常格式不对，OpenCode 用 Schema 验证来"驯服"它。

---

## 为什么需要 JSON 验证？

LLM 的本质是概率生成器，其输出天然不稳定：

| 问题 | 用人话说 |
|---|---|
| JSON 格式错误 | 缺少括号、逗号错位 |
| 字段缺失或类型错误 | 该有的字段没有，类型不对 |
| 产生幻觉字段 | 凭空捏造不存在的字段 |
| 工具参数超出预期范围 | 参数值不合理 |

---

## Effect Schema：类型安全的验证体系

OpenCode 使用 Effect Schema 来对 LLM 生成的输出进行运行时验证。

用人饭店：就像给数据加了一道"安检"——格式不对就过不去。

### 典型模式：TaggedErrorClass

定义结构化的错误类型：

```
ModelNotSelectedError → "SessionRunnerModel.ModelNotSelectedError"
ModelUnavailableError → "SessionRunnerModel.ModelUnavailableError"
```

**用人饭店**：就像给错误贴标签——每种错误有自己的名字，系统能精确识别。

### 数据模型 Schema 验证

```
会话 ID 验证：
  - 必须以 "ses" 开头
  - 品牌化类型（不能当普通字符串用）

权限请求 Schema：
  - 必须包含 id、sessionID、action、resources
  - 可选字段：save、metadata、source
```

**用人饭店**：就像填表格——必填项必须填，选填项可以不填，格式必须正确。

### Schema 解码的核心 API

| API | 用人话说 |
|---|---|
| `Schema.decodeUnknown` | 严格解码：失败就报错 |
| `Schema.decodeUnknownEither` | 返回 Either<错误, 结果> |
| `Schema.decodeUnknownPromise` | 返回 Promise |
| `Schema.decodeUnknownAsync` | 异步解码 |
| `Schema.decodeUnknownEffect` | Effect 解码 |
| `Schema.decodeUnknownOption` | 返回 Option（有或没有） |
| `Schema.parse` | 同时解析和转换 |
| `Schema.asserts` | 类型守卫 |

**用人饭店**：就像不同的"安检方式"——有的严格（失败就报错），有的宽松（返回"有或没有"）。

---

## 工具输入验证链

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

**用人饭店**：就像快递安检——包裹（LLM 输出）过 X 光机（Schema 验证），有问题就退回（返回错误给 LLM 让它重发）。

这种机制保证：即使 LLM 产生格式错误的工具调用，系统也不会崩溃，而是将错误信息反馈给 LLM 让其自我修正。

---

## Permission Schema 验证体系

权限系统有自己的 Schema 层：

| 错误类型 | 用人话说 |
|---|---|
| `DeniedError` | 权限规则明确拒绝（确定性） |
| `RejectedError` | 用户拒绝（人为决策） |
| `CorrectedError` | 用户修改后拒绝（带有反馈信息） |

**用人饭店**：就像三种"拒绝方式"——规则不允许、用户不允许、用户修改后不允许。

### 三层错误设计

- `DeniedError`：权限规则明确拒绝（确定性）
- `RejectedError`：用户拒绝（人为决策）
- `CorrectedError`：用户修改后拒绝（带有反馈信息，可用于 LLM 自我修正）

**用人饭店**：就像三种"拒绝回复"——"不行"、"我不同意"、"改成这样就可以"。

---

## 工具输出验证

工具执行结果也需要 Schema 验证：

```
工具执行结果 → Schema 验证 → 符合格式 → 返回给 LLM
                          不符合格式 → 报错
```

工具注册时，OpenCode 对工具名称进行验证：

```
工具名称 → 验证格式 → 有效 → 注册
                  无效 → 拒绝注册
```

**用人饭店**：就像给工具"上户口"——名字必须合规，不合规则不让注册。

---

## 一句话总结

JSON Schema 验证通过 Effect Schema 对 LLM 输出进行运行时验证，确保格式错误不会导致系统崩溃，而是将错误反馈给 LLM 让其自我修正。

---

*[→ 技术版：查看完整代码实现](../../articles/opencode-model-stability-engineering/02-json-schema-validation.md)*
