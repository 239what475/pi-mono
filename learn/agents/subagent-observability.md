# agents：subagent 子进程的“可观测性”是怎么做的？它和“黑箱 subagent”有什么区别？

日期：2026-02-11  
主题：agents（learn/agents/）  
结论一句话：pi 的 subagent（示例）虽然是子进程，但并不必然是黑箱：它通过 `pi --mode json -p` 把 **AgentSession 事件流**以 JSON Lines 形式持续输出到 stdout；父进程扩展逐行解析这些事件，并把“子进程进度”转换成主会话里 `subagent` 工具的 streaming update（`onUpdate(...)`），因此主 UI 可以看到 subagent 的阶段性输出、tool 结果、错误与 usage——黑箱与否取决于你暴露/消费多少事件，而不是“是否子进程”。

## TL;DR
- “可观测性”在这里指 **运行过程可见**（事件、阶段、工具调用与结果、错误、usage），不是“能看见模型内部思考”。
- subagent 子进程并不黑箱的关键是：**子进程自己把事件流吐出来**（JSONL），父扩展再把这些事件“翻译”为主 UI 可展示的进度更新。
- 你觉得它像黑箱，通常是因为：示例只消费了少量事件（例如 `message_end`/`tool_result_end`），UI 只展示了聚合后的结果；但事件本身并不缺。
- 与某些“黑箱 subagent”差异在于：pi 的 subagent 不是内部不可见实现，而是一个 **可复现、可抓日志、可替换 UI 的 worker 进程**（协议在 stdout）。

## 先定义：这里说的“可观测性”到底是什么
当我们说“subagent 可观测”，通常不是指“能看到模型链路里的隐式推理”，而是指下面这些运行态信号是否能被宿主拿到并展示/记录：
- 子任务是否开始、是否还在跑、卡在哪里
- 子任务当前输出到哪一步（至少是“完成了一条 message/一个工具结果”）
- 子任务调用了哪些工具、工具结果是什么（或至少能看到 toolResult 产物）
- 失败时的 stderr、exit code、stop reason
- 使用统计（turns、tokens、cost）

pi 的 subagent 示例做的是“进程级 worker 的事件可观测”，不是“模型内部白盒”。

## 工作原理（按时间线，纯文字）
下面用官方 `subagent/` 扩展示例解释“子进程可观测”是怎么从 0 到 1 搭起来的。

### Step 1：主 agent 调用 `subagent` 工具（扩展工具）
主会话里，模型会把 `subagent` 当作一个普通 tool 来调用。扩展通过 `pi.registerTool(...)` 注册了这个工具，执行函数签名里包含 `onUpdate` 回调。  
证据：`packages/coding-agent/examples/extensions/subagent/index.ts:408`（注册工具）与 `packages/coding-agent/examples/extensions/subagent/index.ts:420`（`execute(..., onUpdate, ctx)`）。

### Step 2：扩展启动一个新的 `pi` 子进程，并要求它“吐事件流”
subagent 工具在执行时会构造子进程参数，核心点是：
- `--mode json`：输出 JSON 事件流
- `-p`：print 模式（单次执行，跑完退出）
- `--no-session`：不落盘（避免产生子 session 文件；更像一次性 worker）
证据：`packages/coding-agent/examples/extensions/subagent/index.ts:247`（args 里包含 `--mode json -p --no-session`）。

### Step 3：子进程在 JSON mode 下，把 session 事件逐条写到 stdout（JSONL）
子进程跑 print-mode 时，会 `session.subscribe(...)`，并在 JSON mode 下把所有事件 `JSON.stringify` 后 `console.log` 到 stdout。  
这是“可观测性”的根：**事件不是父进程猜出来的，是子进程主动暴露出来的**。  
证据：`packages/coding-agent/src/modes/print-mode.ts:76`（subscribe）与 `packages/coding-agent/src/modes/print-mode.ts:78`（JSON mode 输出所有事件）。

（旁证：CLI 支持 `--mode json`。见 `packages/coding-agent/src/cli/args.ts:68`。）

### Step 4：父扩展逐行解析子进程 stdout，把 JSON event 变成“子任务进度”
subagent 扩展示例里，父进程监听 `proc.stdout`，按 `\\n` 切行，对每行 `JSON.parse(...)`。  
它会筛选感兴趣的事件，把 `event.message` 追加到 `currentResult.messages`（这让父进程能“累积”子进程跑出来的消息/工具结果）。  
证据：`packages/coding-agent/examples/extensions/subagent/index.ts:327`（按行切分 stdout）与 `packages/coding-agent/examples/extensions/subagent/index.ts:294`（`JSON.parse`）以及 `packages/coding-agent/examples/extensions/subagent/index.ts:299`（处理 `message_end`）。

这里你看到的“不是黑箱”的关键点是：父进程拿到的不只是“最终字符串”，而是一串可被增量消费的结构化事件。

### Step 5：父扩展把“子进程进度”翻译回主会话：`onUpdate(...)`
拿到新事件/新 message 后，父扩展会调用 `onUpdate(...)`，把当前累计结果作为 streaming update 回传给主会话（也就是主 UI 里 `subagent` tool output 的实时更新）。  
证据：`packages/coding-agent/examples/extensions/subagent/index.ts:266`（`emitUpdate()` 调用 `onUpdate`）与 `packages/coding-agent/examples/extensions/subagent/index.ts:318`（每次收集到 message 后触发 update）。

这一步是“可观测性桥接”的关键：  
子进程 → stdout(JSONL events) → 父扩展解析 → 主进程 tool streaming update → UI。

### Step 6：结束与错误：stderr/exit code/abort 都是可观测信号
- 父扩展把子进程 `stderr` 聚合到 `currentResult.stderr`（可用于在主 UI 呈现失败原因）。  
  证据：`packages/coding-agent/examples/extensions/subagent/index.ts:334`。
- 退出码被捕获并写入结果。  
  证据：`packages/coding-agent/examples/extensions/subagent/index.ts:338` 与 `packages/coding-agent/examples/extensions/subagent/index.ts:360`。
- abort 会 kill 子进程（避免“子任务在后台偷偷跑完但主 UI 已经停了”的不可控状态）。  
  证据：`packages/coding-agent/examples/extensions/subagent/index.ts:347`。

## 你为什么会觉得“它仍然是黑箱”
你这个直觉很正常，因为“子进程”这个实现方式确实容易让人联想到黑箱；但这里更准确的区分是：

1) **事件暴露粒度**  
示例只消费了子进程事件的一小部分（比如 `message_end`），所以 UI 看到的是“阶段性跳变”，而不是 token-by-token 的流式过程。  
这会让它“像黑箱”，但这是示例的取舍，不是机制的上限。

2) **UI 展示策略**  
即便事件都在，父扩展也可以选择“只展示聚合后的最终输出”。从用户体验看它就像黑箱，但从工程上它仍是“可观测的 worker”。

3) **模型内部思考本来就不可见（或不稳定可见）**  
即使把 `message_update`、tool execution 全展示出来，你仍然看不到“模型内部推理”。这属于 LLM 天生边界，不是 subagent 是否黑箱的核心判据。

## “黑箱 subagent” vs pi 这种 subagent（概念层差异）
如果你看到某些工具把 subagent 称为“黑箱”，常见原因是下面几种（这里不针对某个具体产品，只讲类别差异）：

1) **subagent 是内部实现，不对外暴露事件协议**  
外部只能拿到最终一段文本（或一个总结），看不到它做了哪些工具调用/中间产物，因此是黑箱。

2) **不可复现/不可抓取原始轨迹**  
你无法像运行一个 CLI worker 那样：把 stdout/stderr 抓下来、把每个事件写到日志、或者复跑同一条命令进行调试。

而 pi 的 subagent（示例）是：
- “显式 worker”：就是一个可启动的 `pi` 进程
- “显式协议”：stdout JSON Lines（`--mode json`）就是它对外暴露的事件边界
- “显式桥接”：父扩展把事件转成 `onUpdate`，从而 UI 可见

所以它更像“glass box（事件可见）”，而不是“纯黑箱（只给最终字符串）”。

## 证据锚点（精选）
- `packages/coding-agent/examples/extensions/subagent/index.ts:12`：注释说明使用 JSON mode 捕获结构化输出。
- `packages/coding-agent/examples/extensions/subagent/index.ts:247`：子进程参数 `--mode json -p --no-session`。
- `packages/coding-agent/src/modes/print-mode.ts:76`：print-mode 订阅 session 事件。
- `packages/coding-agent/src/modes/print-mode.ts:78`：JSON mode 把所有事件逐条输出到 stdout。
- `packages/coding-agent/examples/extensions/subagent/index.ts:327`：父进程按行读取 stdout。
- `packages/coding-agent/examples/extensions/subagent/index.ts:294`：逐行 `JSON.parse` 事件。
- `packages/coding-agent/examples/extensions/subagent/index.ts:266`：父进程把子进程进度映射为 `onUpdate(...)`（主 UI 可见的 streaming update）。
- `packages/coding-agent/examples/extensions/subagent/index.ts:334`：聚合 stderr（失败可观测）。

## 边界/未验证点
- 这份笔记基于 `examples/extensions/subagent/` 的参考实现：它选择了“只消费部分事件”来保持实现简单；如果你需要更细粒度（例如 token streaming、tool_execution_* 时间线），需要扩展自行消费更多事件类型。
- 子进程是否“足够透明”，最终取决于：子进程是否输出足够事件 + 父扩展是否展示/记录这些事件。机制提供了通道，但不强制 UI 一定展示全部细节。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)
- [没有内置 subagent，但可以用“pi 调 pi”实现（subagent 扩展示例）](subagents-via-self-invocation.md)

