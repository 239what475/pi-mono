# agents：阅读 packages/agent 源码的起点与路线

日期：2026-02-11
主题：agents（learn/agents/）
结论一句话：阅读 `packages/agent` 源码的最佳起点是 `packages/agent/src/agent.ts` 的 `Agent` 类，因为它是对外的主入口（用户实例化/调用/订阅都从这里开始），并且它把底层 `agentLoop()` 的事件流转成可观测的 state + event；在此基础上再下钻到 `agent-loop.ts`（核心算法与事件序列）和 `types.ts`（整套协议/词汇表），读起来最快建立正确心智模型。

## TL;DR
- 起点：`packages/agent/src/agent.ts`（先看 `prompt()` / `continue()` / `_runLoop()` / `emit()`）。
- 第二站：`packages/agent/src/agent-loop.ts`（看 `runLoop()`、`streamAssistantResponse()`、`executeToolCalls()`）。
- 词汇表：`packages/agent/src/types.ts`（看 `AgentLoopConfig`、`AgentMessage`、`AgentEvent`）。
- 校验理解：`packages/agent/test/agent-loop.test.ts`（看“事件序列应该长什么样”）。

## 为什么把 `agent.ts` 当起点（理由）
1) **它是用户真正使用的入口**  
   使用方一般是 `new Agent(...)` 然后 `agent.prompt(...)`、`agent.subscribe(...)`；从这里读能直接看到 public API、运行期约束（例如 streaming 时不能再次 prompt）和 state 形态。

2) **它展示了“核心抽象”的边界**  
   `Agent` 不做 provider 适配细节，而是把底层 loop 的 `AgentEvent` 事件流消费掉，更新 state，并把事件广播出去。读完这层，你就知道“什么属于 agent-core / 什么属于上层产品（coding-agent）/ 什么属于 ai provider”。

3) **它自然引导你下钻到核心算法**  
   `Agent._runLoop()` 会选择 `agentLoop()` 或 `agentLoopContinue()` 并 `for await` 消费事件；这是最短路径把你带到 `agent-loop.ts` 的真实执行顺序。

## 推荐阅读路线（带目标）

### Step 1：先读 `Agent` 对外 API（10–20 分钟）
文件：`packages/agent/src/agent.ts`
- 目标 A：搞清楚“调用入口”：`prompt()`/`continue()`/`abort()`/`waitForIdle()`
- 目标 B：搞清楚“状态模型”：`AgentState` 的哪些字段会在 streaming 中变化（`isStreaming` / `streamMessage` / `pendingToolCalls` / `error`）
- 目标 C：搞清楚“队列语义”：steering / follow-up 是怎么在 runtime 层生效的（以及为什么不能在 streaming 时直接 prompt）

### Step 2：下钻 loop 算法（20–40 分钟）
文件：`packages/agent/src/agent-loop.ts`
- 目标 A：弄清“事件序列”：为什么会有 `agent_start/turn_start/message_* / tool_execution_* / turn_end / agent_end`
- 目标 B：弄清“LLM 边界”：`transformContext()` → `convertToLlm()` → `streamSimple()`
- 目标 C：弄清“工具执行”：toolCall 如何变成 `tool_execution_*` 与 `toolResult` message
- 目标 D：弄清“多 turn 的原因”：toolCall、steering、follow-up 如何驱动继续循环

### Step 3：回头补齐类型词汇表（随读随查）
文件：`packages/agent/src/types.ts`
- `AgentLoopConfig`：你能在这里看到哪些 hook 会影响 loop 行为（convert/transform/getApiKey/队列等）
- `AgentMessage`：为什么 agent 可以承载自定义 message（声明合并），但 LLM 只吃 `Message[]`
- `AgentEvent`：UI/宿主层应该消费哪些事件类型

### Step 4：用测试文件校验理解（10–20 分钟）
文件：`packages/agent/test/agent-loop.test.ts`
- 目标：对照测试里收集的 events，确认你理解的“事件顺序/边界条件”是否一致（例如无工具时最小事件序列）

## 证据锚点（精选）
- `packages/agent/src/index.ts:1`：对外导出包含 `agent`/`agent-loop`/`types`，说明这些文件就是包的骨架。
- `packages/agent/src/agent.ts:90`（`export class Agent`）：包对外的主入口类。
- `packages/agent/src/agent.ts:314`（`prompt()`）：把输入规范化成 `AgentMessage[]` 并启动 `_runLoop()`（调用入口在这里）。
- `packages/agent/src/agent.ts:383`（`_runLoop()`）：构造 `AgentContext`/`AgentLoopConfig`，并选择 `agentLoop()` 或 `agentLoopContinue()`。
- `packages/agent/src/agent.ts:432`：`for await (const event of stream)` 消费 loop 的事件流，并更新 state + `emit(event)`。
- `packages/agent/src/agent-loop.ts:28`（`agentLoop()`）：新 prompt 的 loop 起点，负责 push 初始生命周期事件。
- `packages/agent/src/agent-loop.ts:104`（`runLoop()`）：两层 while 的核心循环（多 turn、工具、队列都在这里驱动）。
- `packages/agent/src/agent-loop.ts:204`（`streamAssistantResponse()`）：LLM 边界处的 `transformContext`→`convertToLlm`→`streamSimple`。
- `packages/agent/src/agent-loop.ts:294`（`executeToolCalls()`）：工具执行与 `tool_execution_*` / `toolResult` message 的产生逻辑。
- `packages/agent/src/types.ts:22`（`AgentLoopConfig`）：loop 的关键 hook 与队列接口定义（阅读时的“词汇表”）。
- `packages/agent/src/types.ts:129`（`AgentMessage`）：AgentMessage 扩展机制（声明合并）入口。
- `packages/agent/src/types.ts:179`（`AgentEvent`）：统一事件协议定义（UI/宿主消费的核心）。
- `packages/agent/test/agent-loop.test.ts:84`：测试里验证 `agentLoop()` 的基础事件序列（理解校验点）。

## 边界/未覆盖
- 本笔记只给出“读源码起点与路线”，不会展开 provider 侧（`@mariozechner/pi-ai`）的具体 streaming 事件细节；它们在 agent-loop 里被统一映射为 `message_update`。
- `packages/coding-agent` 的 `AgentSession` 会在 `Agent` 之上加一层产品能力（session persistence/compaction/extensions 等）；如果你的目标是理解 pi 产品而不是 agent-core，需要再读 `AgentSession`。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)

