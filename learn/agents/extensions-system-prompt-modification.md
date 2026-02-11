# agents：extensions 如何修改 system prompt？

日期：2026-02-11  
主题：agents（learn/agents/）  
结论一句话：extensions 可以在 `before_agent_start` 事件里“替换本次 turn 的 system prompt”；此外还可以通过 `resources_discover` 增量扩展 skills/contexts 等资源，间接导致 base system prompt 重建，从而更持久地影响后续 turns 的 system prompt 组成。

## TL;DR
- **能改，但语义是“替换字符串”**：扩展在 `before_agent_start` 返回 `systemPrompt`，宿主会把它设为本次 turn 使用的 system prompt。
- **作用域是“按 turn”**：宿主每次发送用户 prompt 前都会用 base system prompt 作为起点；若本次没有扩展修改，会显式恢复到 base（避免上次残留）。
- **多扩展是链式覆盖**：多个扩展都返回 `systemPrompt` 时，后注册/后执行的 handler 会看到前一个的修改结果，并继续替换（最终“最后一次替换”的结果生效）。
- **还有一种“更持久”的间接影响**：扩展通过 `resources_discover` 添加 skills/context/prompt 等资源路径 → 宿主扩展资源并重建 base system prompt → 后续 turns 的基线 system prompt 组成会变化。

## 先分清：base system prompt vs “本次 turn 的 system prompt”
在 pi 的实现里，system prompt 可以理解为两层：

1) **base system prompt（基线）**  
由宿主根据资源系统构建：默认系统提示词（或 `SYSTEM.md` 自定义提示词）+ `APPEND_SYSTEM.md` + 项目 context files + skills + 当前工作目录/时间等。

2) **effective system prompt（本次 turn 真正用的）**  
通常等于 base；但在本次 turn 开始前，扩展可以把它替换成自定义版本（例如追加企业策略、注入审计要求、为某类 prompt 临时切换到另一套系统提示词）。

## extensions 修改 system prompt 的主路径：before_agent_start
这是“直接改 system prompt 字符串”的路径，语义清晰、作用域明确。

### 触发时机（在 prompt 链路中的位置）
当用户输入一条 prompt 时，宿主的顺序大致是：
1. 若输入以 `/` 开头：先尝试执行 extension command（命中则不进入 LLM turn）
2. 若有 `input` handlers：先发 `input` 事件（发生在 skills/prompt template 展开之前，可改写/吞掉输入）
3. 展开 `/skill:` 与 prompt templates（把文本扩成实际给模型看的内容）
4. **发 `before_agent_start`**：这是进入 agent loop 之前的最后窗口  
   - 扩展能注入 custom messages
   - 扩展能替换 system prompt（本次 turn）
5. 调用 agent-core 开始流式对话与工具调用

### “替换”语义与链式覆盖
`before_agent_start` 的 handler 并不是给你“patch 某几段”的差异格式，而是直接返回一个新的 system prompt 字符串。

多扩展情况下：
- runner 会把当前 system prompt 作为 event.systemPrompt 传给每个 handler
- 如果某个 handler 返回了 `systemPrompt`，runner 会把它当作“新的 currentSystemPrompt”
- 后续 handler 看到的就是更新后的版本（所以称为“chained”）

这让你可以做两种风格：
- **完全替换**：忽略旧值，返回一套固定 system prompt
- **基于旧值拼接**：读取 event.systemPrompt，在它上面追加/插入内容后返回（实现“可组合”的扩展策略）

### 生效范围：一轮 turn 内有效，下一轮会自动回到 base
宿主在每次 prompt 时都会把 base system prompt 当作起点传给 `before_agent_start`。并且当本轮没有扩展返回 `systemPrompt` 时，会显式把 agent 的 system prompt 重置回 base（避免上次替换残留影响下一轮）。

因此可以把它理解为：
> `before_agent_start` 是“本轮 turn 的 system prompt 过滤器/覆写器”，而不是永久全局配置。

## 更持久的影响路径：resources_discover → 重建 base system prompt
除了 per-turn 替换，扩展还可以通过 `resources_discover` 让宿主“把更多资源纳入资源系统”：
- 额外 skills（通常会被格式化后追加到 system prompt）
- 额外 prompt 资源（影响 slash commands 与系统能力表述）
- 额外 themes（UI 层面）

宿主在 `session_start` 后会触发资源发现，把扩展返回的路径扩展进资源系统，然后重建 base system prompt 并设置到 agent 上。

这个路径的特点：
- **更持久**：它改变的是“基线构建输入”，之后每一轮都以新的 base 为起点
- **更结构化**：适合把能力做成“可配置资源”，而不是把一大段纯文本硬塞进 system prompt
- **更安全可审计**：资源文件可被版本控制、可 review（相比在 handler 里拼字符串）

## 常见用法场景（纯文字例子）
- 企业/团队规范：在 `before_agent_start` 里按目录/仓库类型选择不同 system prompt 模板（例如后端/前端/基础设施），本轮强制执行。
- 安全与合规：在 `before_agent_start` 里追加“禁止泄露密钥/脱敏规则/审计标记”等；在 `tool_call` 里阻断危险工具调用形成闭环。
- 工作流增强：通过 `resources_discover` 分发一组团队 skills（统一的操作手册与步骤模板），让 base system prompt 自动携带这些技能索引。

## 证据锚点（精选）
- `packages/coding-agent/src/core/extensions/types.ts:483`（`BeforeAgentStartEvent`）：事件包含 `systemPrompt` 字段，说明扩展能看到当前 prompt 与 system prompt。
- `packages/coding-agent/src/core/extensions/types.ts:789`（`BeforeAgentStartEventResult`）：结果包含 `systemPrompt`，注释说明“替换本轮 system prompt，多扩展 chained”。
- `packages/coding-agent/src/core/extensions/runner.ts:690`（`emitBeforeAgentStart()`）：runner 逐扩展逐 handler 传递 `currentSystemPrompt`，并在返回值里更新它。
- `packages/coding-agent/src/core/agent-session.ts:654`（`prompt()`）：展示 prompt 处理顺序（command → input → 展开 → before_agent_start → agent）。
- `packages/coding-agent/src/core/agent-session.ts:762`（调用 `emitBeforeAgentStart`）：宿主把 base system prompt 传入扩展链路作为起点。
- `packages/coding-agent/src/core/agent-session.ts:782`（setSystemPrompt）：有扩展替换则设置替换值；否则重置回 base（避免上轮残留）。
- `packages/coding-agent/src/core/agent-session.ts:1736`（`extendResourcesFromExtensions()`）：扩展资源后会 `extendResources` 并重建 base system prompt。
- `packages/coding-agent/src/core/agent-session.ts:622`（`_rebuildSystemPrompt()`）：base system prompt 来自资源系统（system/append prompt、skills、context files）并经统一 builder 构建。
- `packages/coding-agent/src/core/system-prompt.ts:35`（`buildSystemPrompt()`）：说明 system prompt 的组成方式（自定义 prompt/追加 prompt/context/skills/时间与目录）。

## 边界/注意点
- `before_agent_start` 是在“用户 prompt”路径上的 hook；如果扩展以其它方式触发 turn（例如某些内部触发路径），未必都会走到同一个 hook（需要看具体调用栈）。
- 多扩展替换 system prompt 时，最终结果依赖 handler 执行顺序；若想可组合，扩展应尽量基于 event.systemPrompt 做追加而不是完全覆盖。
- system prompt 是高权限控制面：把它交给不可信扩展等同把 agent 行为控制权交出去；应配合扩展来源治理与审计策略。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)
- [extensions（扩展）机制：设计与实现](extensions-design-and-implementation.md)

