# agents：一次 prompt 的完整事件链（输入→LLM→工具→UI）

日期：2026-02-11
主题：agents（learn/agents/）
结论一句话：一次 prompt 的主干链路是 **UI 输入 → `AgentSession.prompt()`（产品层预处理）→ `Agent.prompt()`（runtime）→ `agentLoop()`（协议/流程）→ `pi-ai streamSimple()`（provider streaming）→ tool execution → 事件流回到 `AgentSession.subscribe()` → UI 渲染**；整个系统用 `AgentEvent`（message/turn/tool 生命周期）把“LLM + 工具 + UI”解耦。

## TL;DR
- UI（InteractiveMode）只做 I/O：输入→调用 `session.prompt()`；输出→订阅 `session.subscribe()` 并按事件类型渲染。
- `AgentSession` 是“产品层核心”：统一处理扩展（extensions）、消息持久化（sessionManager）、队列（steer/followUp）、以及把 `AgentEvent` 转发给 UI。
- `Agent` 是“runtime”：负责把 `agentLoop()` 产生的事件更新到内部 state，并把事件广播给订阅者。
- `agentLoop()` 是“协议/流程”：负责 turn、消息生命周期事件、LLM streaming 到 `message_update` 的映射，以及工具执行的 `tool_execution_*` 事件。
- `pi-ai` 提供 `streamSimple()`：把 model+context 路由到具体 provider 的流式实现。

## 工作原理（流程图）

```text
用户输入（TUI Editor）
  ↓
InteractiveMode -> AgentSession.prompt(text)
  ↓
AgentSession 组装 AgentMessage[]（含图片/扩展注入/模板展开）
  ↓
Agent.prompt(messages) -> Agent._runLoop()
  ↓
agentLoop()/agentLoopContinue()
  ↓
streamAssistantResponse():
  AgentMessage[] --transformContext--> AgentMessage[]
              --convertToLlm--> Message[] (LLM)
  ↓
pi-ai streamSimple() -> provider.streamSimple() -> AssistantMessageEventStream
  ↓
AgentEvent:
  message_start/update/end + tool_execution_* + turn_* + agent_*
  ↓
Agent 更新 state + emit(event)
  ↓
AgentSession 内部处理（extensions + persistence）+ 转发给 UI
  ↓
InteractiveMode.handleEvent(event) 更新组件并 requestRender()
```

## 一步一步（从按下 Enter 开始）

下面以 interactive mode 为例，按“发生顺序”拆开每一层做的事；你可以把它当作一次 prompt 的“执行栈”。

### Step 1：TUI 收到输入并决定走哪条分支（normal / streaming / compaction / bash）
- 在 interactive mode 里，用户提交文本后会进入一段“分支逻辑”：
  - bash：以 `!` 开头走 bash 执行路径（不是普通 prompt）。
  - compaction：正在 compact 时，普通输入会被排队（extension command 仍可立即执行）。
  - streaming：如果 agent 正在 streaming，本次输入不会立刻触发新的 LLM call，而是通过 `streamingBehavior` 入队（steer 或 followUp）。
  - normal：否则走普通 prompt（进入 `AgentSession.prompt()`）。

为什么需要这层分支：因为 **`Agent.prompt()` 在 streaming 时会直接抛错**（agent-core 设计上要求用 steer/followUp queue）。因此产品层（coding-agent）必须在 UI 层就把输入导向“入队”或“新 prompt”。

### Step 2：`AgentSession.prompt()` 做产品层预处理（扩展/模板/队列/校验）
`AgentSession.prompt(text)` 并不是“直接把 text 发给 LLM”，它会依次做：

1) **extension command（`/xxx`）优先**
- extension command 会立即执行并自行决定是否触发 LLM（不会走普通 prompt）。

2) **extensions input hook（可拦截/transform 输入）**
- extensions 可以把输入标记为 handled（完全接管），或 transform（改写 text / images）。

3) **展开 skill / prompt template**
- 把用户输入的短指令展开成完整提示词块（skill block / template 内容）。

4) **Streaming 时入队**（关键）
- 如果 `this.isStreaming === true`：不会调用 `agent.prompt()`，而是 `_queueSteer()` 或 `_queueFollowUp()`。
- steer：用于“打断/改道”，会在工具执行间隙注入，并跳过剩余工具。
- followUp：用于“等当前工作做完再说”，只有在 agent 将要停止时才会被注入成下一轮 turn。

5) **非 streaming：校验 model + api key + compaction check**
- 没 model / 没 key 会直接 throw（让 UI 显示错误/引导登录）。
- 在发送 prompt 之前，会检查是否需要 compaction（尤其是上一次 assistant 可能 aborted 但仍占上下文）。

6) **构造本次 prompt 的 `AgentMessage[]`**
- 至少包含一条 `user` message（文本 + 可选图片），并可把一些 pending messages（nextTurn asides / extensions 注入的 custom messages）一起拼进去。

7) **extensions before_agent_start（注入 custom messages + 修改 system prompt）**
- extensions 有机会在“真正调用 agent”之前注入 custom messages，或者临时修改 system prompt（每 turn 可变）。

8) **调用 runtime：`await this.agent.prompt(messages)`**
- 从这里开始进入 agent-core 的事件流世界。

### Step 3：`Agent.prompt()` 规范化输入并启动 `_runLoop()`
- `Agent.prompt()` 的第一职责是把输入规范化为 `AgentMessage[]`（字符串会被包成 `user` message；数组则直接使用）。
- 然后进入 `_runLoop()`：创建 AbortController、设置 `isStreaming=true`，构造 `AgentContext` 与 `AgentLoopConfig`，并启动 `agentLoop()`（或 `agentLoopContinue()`）。

### Step 4：`agentLoop()` 先发“生命周期事件”，再进入 `runLoop()` 核心循环
对于“新 prompt”入口 `agentLoop(prompts, ...)`：
- 立刻 push：`agent_start` → `turn_start`
- 对每条 prompt message：push `message_start` → `message_end`
- 然后 `runLoop(...)` 负责后续所有 turn/工具/继续对话

这一步很重要：UI 看到的用户消息并不是“自己渲染出来的”，而是来自 agentLoop 统一事件流（这样 JSON/RPC 模式也可以重放同样的事件序列）。

### Step 5：`runLoop()` 的两层 while：为什么一次 prompt 可能跑多个 turn
`runLoop()` 可以把一次 prompt 扩展成多个 turn，原因有两个：
1) assistant message 里包含 toolCall → 执行工具 → 把 toolResult 注入上下文 → 再跑一轮 assistant 响应（新的 turn）
2) followUp queue 在 agent “要结束时”注入 → 继续跑后续 turn

因此 turn 的定义更准确是：**一条 assistant 响应 +（可选）若干 toolCall 执行 + toolResults**。

核心步骤（每轮 turn）：
1) 注入 pending messages（steer/followUp 变成的 user messages）
2) `streamAssistantResponse()`：调用 LLM 并把 provider streaming 映射为 `message_update`
3) 若 assistant content 含 toolCall：`executeToolCalls()` 逐个执行，产出 `tool_execution_*` + `toolResult message_*`
4) push `turn_end`
5) 检查 steering messages（可能跳过剩余工具并触发下一轮）
6) 当没有 toolCall 且没有 steering：检查 followUp queue；没有则最终 push `agent_end`

### Step 6：`streamAssistantResponse()`：LLM 边界的两次转换 + streaming 映射
这是“AgentMessage 世界”和“LLM Message 世界”的分界线：
1) `transformContext(messages)`：可选，对 AgentMessage 做裁剪/注入
2) `convertToLlm(messages)`：必选，把 AgentMessage 过滤/转换成 LLM 接受的 `Message[]`
3) 构造 `llmContext = { systemPrompt, messages: llmMessages, tools }`
4) 解析 apiKey（通过 `getApiKey()`，支持 OAuth token 刷新等）
5) 调用 `streamFunction`（默认 `pi-ai streamSimple()`）拿到 provider 的事件流
6) 把 provider 的事件统一映射为 agent 的 message 事件：
   - provider `start` → `message_start`（assistant partial）
   - provider `text_delta/thinking_delta/toolcall_delta...` → `message_update`
   - provider `done/error` → `message_end`（assistant final）

### Step 7：`executeToolCalls()`：工具事件 + toolResult 消息是怎么产生的
对每一个 toolCall：
1) push `tool_execution_start`（带 toolCallId/toolName/args）
2) validate args（schema 校验）
3) `tool.execute(id, args, signal, onUpdate)`：
   - onUpdate 回调会 push `tool_execution_update`
4) push `tool_execution_end`（带最终 result + isError）
5) 把 result 包成一条 `toolResult` message，并 push `message_start` + `message_end`
6) 询问 steering queue：若用户有新输入，则跳过剩余工具（为每个“被跳过的 toolCall”生成一个 error toolResult）

### Step 8：`Agent` 把事件流变成 state，并 emit 给订阅者
`Agent` 在 `for await (const event of stream)` 中做两件事：
1) 更新 internal state（例如 `streamMessage`、`pendingToolCalls`、`isStreaming`）
2) 把 event 广播给订阅者（UI/Session）

这意味着：UI 不需要“自己拼状态”，只需要消费事件即可；同时，state 也可供 UI 在需要时读取（例如 footer token/模型信息）。

### Step 9：`AgentSession` 再做一次“产品层处理”，然后把事件转发给 UI
`AgentSession` 在内部订阅 `agent.subscribe(this._handleAgentEvent)`，用于：
- extensions：把 agent 的 turn_start/turn_end 等转成 extension 事件（并保证顺序：先扩展再 UI）
- persistence：在 `message_end` 时把 user/assistant/toolResult 写入 `sessionManager`
- retry/compaction：在 `agent_end` 后触发自动重试与自动 compact（不属于 agent-core 的职责）
- queue book-keeping：在 `message_start(user)` 时从 steering/followUp 列表里移除已投递的文本，保证 UI 看到的队列状态一致

最终：`AgentSession.subscribe()` 的监听者（InteractiveMode）会收到同一条事件（加上一些 session 扩展事件，例如 auto_compaction_*）。

### Step 10：InteractiveMode 把事件映射为“组件树变化”
InteractiveMode 的 `handleEvent()` 大致是：
- `agent_start`：启动 loader（Working...），清理状态区
- `message_start(user)`：把用户消息组件插到 chat；刷新 pending messages 区
- `message_start(assistant)`：创建 streaming assistant 组件并插入 chat
- `message_update(assistant)`：更新 streaming 组件；如果 assistant 内容里出现 toolCall，则提前创建 ToolExecutionComponent（用于显示 tool args）
- `tool_execution_start/update/end`：更新 tool 组件的执行状态与输出
- `message_end(assistant)`：结束 streaming；若 aborted/error，则把所有 pending tool 组件置为 error
- `agent_end`：停 loader、清理 streaming 状态、触发 shutdown 检查

## 事件时间线示例（包含一次工具调用）

假设用户输入 “列出当前目录文件”，模型决定调用一个工具，然后再根据工具结果回复：

```text
agent_start
turn_start
  message_start(user)  -> UI 显示用户消息
  message_end(user)
  message_start(assistant) -> UI 创建 streaming assistant 组件
  message_update(assistant) x N
    - 可能出现 toolCall delta -> UI 提前创建 ToolExecutionComponent（显示 args）
  message_end(assistant) -> UI 固化 assistant 内容

  tool_execution_start -> UI 标记 tool 开始执行
  tool_execution_update x N (可选) -> UI 流式更新 tool 输出
  tool_execution_end   -> UI 固化 tool 输出（成功/失败）
  message_start(toolResult)
  message_end(toolResult)
turn_end

turn_start
  message_start(assistant)
  message_update x N
  message_end(assistant)
turn_end
agent_end
```

## `toolCall`（assistant 内容）与 `tool_execution_*`（执行状态）的关系
- `toolCall` 出现在 assistant message 的 content 里：代表“模型决定要调用什么工具、参数是什么”（可能是 partial，直到 `toolcall_end` 才完整）。
- `tool_execution_*` 代表“本地/宿主真正开始执行该工具、执行进度、最终结果”。
- UI 同时消费两条信号是合理的：
  - toolCall 让 UI 提前展示“将要执行的工具 + 参数”
  - tool_execution_* 让 UI 展示“正在执行/输出流/完成状态”

## 关键点展开

### 1) 输入从哪里进来（InteractiveMode）
- Interactive 模式里，用户提交文本最终会走到 `session.prompt(...)`（若正在 streaming，会用 `streamingBehavior` 决定是 steer 还是 followUp）。
- 该层不会直接调用 provider，也不会自己跑工具；它只订阅事件并更新组件树。

### 2) `AgentSession.prompt()` 做了哪些“产品层”的事
- 处理 extension command（`/xxx`）的“立即执行”路径（不会进入普通 prompt）。
- 允许 extensions 拦截/转换 input（例如改写 text、附加图片）。
- 展开 skill/template（让“输入文本”变成“实际提示词块”）。
- 当 agent 正在 streaming 时：不会直接 prompt，而是入队（steer/followUp），并在 UI 上反映队列。
- 当不在 streaming 时：校验 model 与 API key，构造 `AgentMessage[]`，再调用 `agent.prompt(messages)`。

### 3) `Agent` 如何把 loop 事件变成 state + 对外事件
- `Agent` 将“底层 loop”抽象成统一的 `AgentEvent` 事件流；一边更新自己的 state（`isStreaming`、`streamMessage`、`pendingToolCalls` 等），一边把事件 emit 给订阅者（UI/Session）。

### 4) LLM streaming 事件如何变成 `message_update`
- `agentLoop.streamAssistantResponse()` 在 LLM 边界处做两次转换：
  1) `transformContext()`：AgentMessage 层面的裁剪/注入（可选）
  2) `convertToLlm()`：把 AgentMessage 变成 LLM 接受的 Message[]
- 然后通过 `streamSimple()` 获取 provider 事件流，并把 text/thinking/toolcall 的 delta 统一映射成 `message_update` 事件（携带 `assistantMessageEvent` + 当前 partial message）。

### 5) 工具执行如何进入事件链（tool_execution_* + toolResult message）
- `agentLoop` 会从最终 assistant message 中收集 toolCall，逐个执行：
  - 执行前发 `tool_execution_start`
  - 工具可流式更新：通过 callback 发 `tool_execution_update`
  - 执行结束发 `tool_execution_end`
  - 然后把结果包装成一条 `toolResult` message，并发 `message_start/message_end`
- 如果在工具执行后检测到 steering 消息，会跳过剩余工具并注入“skipped”结果。

### 6) UI 如何消费事件
- `InteractiveMode` 订阅 `AgentSessionEvent`：
  - `agent_start/agent_end` 控制 loader、状态区
  - `message_start/update/end` 更新聊天消息（尤其是 assistant streaming 过程）
  - `tool_execution_*` 更新 tool output 组件
  - 这使得 UI 不需要理解 provider 细节，只要理解统一事件协议即可。

## 证据锚点（精选）
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts:1989`：streaming 时提交输入会调用 `session.prompt(text, { streamingBehavior: "steer" })`（输入→产品层入口）。
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts:2078`：`message_update(assistant)` 时 UI 更新 streaming message，并从 content 中识别 toolCall 以提前创建 tool 组件（toolCall→UI）。
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts:2150`：UI 消费 `tool_execution_start/update/end` 更新 tool 组件（执行状态→UI）。
- `packages/coding-agent/src/core/agent-session.ts:654`：`AgentSession.prompt()` 处理扩展拦截、模板/skill 展开、streaming 入队、校验 model/apiKey、构造 user message（产品层预处理）。
- `packages/coding-agent/src/core/agent-session.ts:791`：最终调用 `this.agent.prompt(messages)` 进入 agent-core runtime（产品层→runtime）。
- `packages/coding-agent/src/core/agent-session.ts:310`：`_handleAgentEvent` 先发 extension 事件，再 `_emit` 给 UI，并在 `message_end` 进行持久化与自动处理（runtime 事件→产品层）。
- `packages/agent/src/agent.ts:432`：`Agent` 消费 `agentLoop()` 事件流，更新 state 并 `emit(event)`（loop→runtime state/events）。
- `packages/agent/src/agent-loop.ts:44`：`agentLoop()` 启动时 push `agent_start/turn_start` + prompt 的 `message_start/message_end`（生命周期开头）。
- `packages/agent/src/agent-loop.ts:211`：LLM 边界处 `transformContext`→`convertToLlm`→`streamFunction(streamSimple)`（AgentMessage→LLM Message）。
- `packages/agent/src/agent-loop.ts:242`：把 provider streaming 事件映射成 `message_start/message_update/message_end`（provider→AgentEvent）。
- `packages/agent/src/agent-loop.ts:309`：工具执行产生 `tool_execution_*`，并包装成 `toolResult` message_*（tools→AgentEvent）。
- `packages/ai/src/stream.ts:44`：`streamSimple()` 路由到具体 provider 的 streaming 实现（ai 层路由）。

## 反例/边界情况
- 不同 mode（print/json/rpc/sdk）会消费同一套核心事件，但“输出层”不同：例如 JSON mode 可能直接把事件流打印为 JSONL，而不是渲染 TUI。这里主要以 interactive 为例。
- extensions 可以拦截 input、修改 system prompt、注入 custom message，这会改变“用户看到/模型收到”的内容，但仍应遵循同一事件链。
- streaming 中的用户输入不会立刻变成一个新的 LLM turn；而是入队（steer/followUp）并在合适的时机注入（可能导致跳过剩余工具）。

## 相关内容（延展沉淀）
- 入口：
  - `packages/coding-agent/src/modes/interactive/interactive-mode.ts:1977`：输入分支（bash/compaction/streaming/normal）。
- 配置：
  - `packages/coding-agent/src/core/sdk.ts:287`：创建 `Agent` 并注入 `convertToLlm/transformContext/getApiKey/steeringMode/followUpMode` 等关键钩子。
- 测试/示例：
  - `packages/agent/test/agent-loop.test.ts:84`：验证 `agentLoop()` 会产出基础事件序列（agent_start/turn_start/message_*/turn_end/agent_end）。
- 失败模式：
  - `packages/agent/src/agent-loop.ts:144`：assistant stopReason 为 `error/aborted` 时会提前 `turn_end` + `agent_end`。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)

## 后续问题
- toolCall 在 streaming message_update 中出现 vs `tool_execution_*` 事件的关系：UI 目前同时消费两条信号，是否存在重复/竞态需要单独梳理。
- JSON/RPC mode 的事件输出与 interactive 的消费是否完全同构（有无额外的 session 事件，如 auto_compaction/auto_retry）。
