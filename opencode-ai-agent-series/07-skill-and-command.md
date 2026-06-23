# 07 - 技能系统与命令系统

## 编写 TODO

### 需要阅读的源码

- [ ] 深入阅读 `packages/core/src/skill.ts`（160 行，完整读完）
- [ ] 阅读 `packages/core/src/skill/discovery.ts` — Skill 发现机制
- [ ] 阅读 `packages/core/src/config/markdown.ts` — Markdown frontmatter 解析
- [ ] 阅读内置命令文件（在 `.opencode/command/` 或 `packages/opencode/src/command/` 下）
- [ ] 阅读 `packages/core/src/tool/skill.ts` — skill 加载工具的实现

### 分析要点

- [ ] **Skill 的三类来源**: 
   - `DirectorySource`（type: "directory"）— 扫描本地 `{*.md, **/SKILL.md}`
   - `UrlSource`（type: "url"）— 远程拉取
   - `EmbeddedSource`（type: "embedded"）— 直接在配置中内嵌
- [ ] **发现流程**: 配置 → SkillDiscovery → glob 扫描 → 解析 frontmatter → 缓存加载 → Agent 请求时权限过滤
- [ ] **去重逻辑**: `Source.equals()` 方法。`embedded` 类型只比较 `name`，不比较 `content`——这意味着同名不同内容的 embedded skill 被视为同一个
- [ ] **权限过滤**: `available()` 函数—`PermissionV2.evaluate("skill", skill.name, agent.permissions).effect !== "deny"`
- [ ] **命令结构**: 每个命令一个 MD 文件，含 YAML frontmatter（description / model / subtask）+ Markdown body + `$ARGUMENTS` 占位符

### 深度洞察

- `State.create<Data, Draft>` 模式：`source()` 是写操作（追加 source 到 draft 数组），`list()` 是读操作（返回不可变视图）。但实际持久化是通过 `SkillV2.load()` 完成的——draft 的作用主要是给 transform 函数使用的
- `embedded` 类型的去重只用 `name`——不看 `content`。这意味着如果两个不同地方注册了同名但内容不同的 embedded skill，只保留第一个。这可能不是 bug——name 作为标识符确实应该唯一——但 content 冲突时静默去重而不报错，可能会导致困惑
- 命令文件中的 `$ARGUMENTS` 是纯字符串替换——最朴素的参数注入。相比模板引擎（如 Liquid/Handlebars），这限制了表达能力（不能做循环/条件），但也避免了模板的复杂性和安全风险
- 命令的 `model` 字段允许为每个命令指定不同的模型。例如 `changelog.md` 用 gpt-5.4、`translate.md` 用 claude-opus-4-8。这是一个"任务-模型匹配"优化——复杂任务用好模型，简单任务用便宜模型

### 依赖

- [ ] 02 - Agent 系统（权限评估）
