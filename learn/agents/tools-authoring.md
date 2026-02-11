# agents：工具（tools）是谁写的？agent 能不能“自己写工具”

日期：2026-02-11
主题：agents（learn/agents/）
结论一句话：在 `packages/agent` 里，**工具是宿主/开发者提供的 `AgentTool`（带 `execute()` 函数）**，agent/LLM 只能“调用已注册的工具”，不能在运行时凭空创建一个新工具并让宿主执行；在 `packages/coding-agent`（pi 产品）里，你可以通过 **extensions** 用 `pi.registerTool()` 注册新工具，模型可以“帮你生成扩展代码”，但是否加载/启用工具仍由宿主决定。

## TL;DR
- “agent 自己写工具”如果指 **LLM 运行时动态发明并执行新工具**：不支持；工具必须预先注册，否则会报 `Tool <name> not found`。
- 如果指 **你（开发者）给 agent 增加自定义工具**：支持；在 `packages/agent` 用 `AgentTool` + `agent.setTools()`。
- 如果指 **在 pi 里扩展工具**：支持；写一个 extension 调 `pi.registerTool()`，然后加载这个 extension。
- 模型最多能“写出工具的代码/扩展文件”，但“把它变成可执行的 tool”仍需要宿主注册（安全边界）。

## 1) `packages/agent` 的工具到底是什么（host-defined code）
在 agent-core 中，工具的核心类型是 `AgentTool`：它不是纯描述信息，而是**带执行函数的 TypeScript 对象**。

- `AgentTool` 继承自 `Tool`，并额外要求实现 `execute(...)`，返回结构化的 `AgentToolResult`。这意味着工具本质上是宿主代码的一部分，而不是模型“发明出来就能跑”的东西。  

## 2) 为什么说“不能凭空写工具”：执行时必须能在 registry 里找到
工具调用发生在 `agentLoop` 内部执行阶段：

1. 模型在 assistant message 中产出 `toolCall`（包含 `name` 与 `arguments`）。
2. `executeToolCalls()` 会用 `tools?.find((t) => t.name === toolCall.name)` 在当前上下文的 tools 列表里查找。
3. 找不到就抛错 `Tool <name> not found`，并把错误作为 `toolResult`（`isError: true`）回灌给模型。

因此：**模型不能凭空声明“我现在有一个新工具 X”并让 runtime 执行**；你必须把工具对象放进 `context.tools` / `agent.state.tools`。

## 3) 你如何“自己写工具”（agent-core 级别）
`packages/agent` 官方 README 里给出最直接的写法：
- 定义一个 `AgentTool`（`name/label/description/parameters/execute`）
- 调用 `agent.setTools([readFileTool])` 注册到 agent 上

执行时：
- 参数会被 schema 校验（工具的 `parameters` 是 TypeBox schema）
- 工具内部失败应该 `throw`，agent-loop 会捕获并标记 `isError: true`
- 工具也可以通过 `onUpdate` 流式报告进度（对应 `tool_execution_update` 事件）

## 4) 在 pi（coding-agent）里扩展工具：extensions 的 `pi.registerTool()`
如果你说的“agent 会自己写工具”是指“pi 可以加新工具”，那通常指的是 **extensions 机制**：

- extension API 提供 `pi.registerTool(definition)`，把工具暴露给 LLM 调用。
- 在 RPC mode 下也能用（会把 extension UI 请求映射成 JSON 协议），但没有 TUI 的一些 UI 能力。

这层是 `packages/coding-agent` 的能力，不是 `packages/agent` 自带的。

## 5) 安全边界：模型能写代码 ≠ 模型能自动获得新权限
在 pi 里，模型确实能用已有工具（比如 `write/edit/bash`）生成一份 extension/tool 的源码文件。
但这仍然是“生成代码”，不是“自动注册并执行新工具”：
- 你必须显式加载/启用 extension（或把工具注入到运行时）
- 否则 runtime 依然只认当前工具列表

这也是合理的安全设计：否则模型可以“自我扩权”。

## 证据锚点（精选）
- `packages/agent/src/types.ts:157`（`AgentTool`）：工具包含 `execute(...)`，本质是宿主代码。
- `packages/agent/src/agent-loop.ts:307`：工具执行时从 `tools` 列表按 `name` 查找（不是动态生成）。
- `packages/agent/src/agent-loop.ts:320`：找不到工具会抛错 `Tool <name> not found`。
- `packages/agent/src/agent-loop.ts:322`：`validateToolArguments(...)`：工具参数会被 schema 校验。
- `packages/agent/src/agent-loop.ts:324`：调用 `tool.execute(...)`，并用 callback 产生 `tool_execution_update` 事件。
- `packages/agent/src/agent-loop.ts:333`：工具执行异常会被捕获并标记 `isError = true`，作为 toolResult 回灌。
- `packages/agent/README.md:319`：README 展示如何定义 `AgentTool` 并 `agent.setTools(...)`（工具是你写的）。
- `packages/agent/test/agent-loop.test.ts:256`：测试里把 `tools: [tool]` 放进 context，验证 tool_execution 事件会产生。
- `packages/coding-agent/docs/extensions.md:836`：extension API 文档明确提供 `pi.registerTool(definition)`。
- `packages/coding-agent/src/core/extensions/types.ts:897`：类型层声明 `registerTool(...)` 是 extension API 的一部分。
- `packages/coding-agent/src/core/extensions/loader.ts:150`：`registerTool` 会把工具写入 extension 的工具集合（注册机制在宿主侧）。

## 边界/未验证点
- “工具能不能在运行时热插拔”取决于上层（例如 coding-agent 的 extensions reload/registry 策略），agent-core 本身只消费 `tools` 列表并执行。
- 本笔记没有展开 coding-agent 的 tool registry 如何与 system prompt 绑定（只说明了 register/execute 的边界）。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)
- [一次 prompt 的完整事件链（输入→LLM→工具→UI）](prompt-to-ui-event-chain.md)

## 后续问题
- 想“安全地让模型帮你写扩展工具”：应该配什么审核/权限策略（例如只允许读、禁止 bash 等）？
- coding-agent 的 tools registry 与 system prompt 的绑定策略（工具列表如何影响模型可用工具与提示词）是否需要单独梳理？

