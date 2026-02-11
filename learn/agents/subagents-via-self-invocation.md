# agents：pi 没有内置 subagent，但可以用“pi 调 pi”实现（subagent 扩展示例）

日期：2026-02-11  
主题：agents（learn/agents/）  
结论一句话：pi 的核心（尤其是 `packages/coding-agent`）明确选择 **不内置 sub-agents**；所谓“自己调用自己”通常指通过 extensions 或脚本在宿主进程里 **spawn 独立的 `pi` 子进程**（如官方 `subagent/` 扩展示例），用子进程隔离上下文窗口，并通过 `--mode json` 的事件流把结果回传给主会话。

## TL;DR
- **没有内置 subagent**：pi 不提供一套“子代理对象/调度器”的内建机制；多代理是你自己的工作流选择（tmux、多进程、扩展、第三方 package）。
- **“自己调用自己”= 组合进程**：`subagent/` 扩展示例每次委派任务都会启动一个新的 `pi` 进程（`pi --mode json -p --no-session ...`），这就是“pi 调 pi”。
- **隔离来自进程边界**：每个子进程有独立的消息历史与 token/context window，不会污染主会话；`--no-session` 让它不落盘。
- **通讯靠 JSON 事件流**：子进程在 JSON 输出模式下把 `AgentSession` 事件逐行输出；父扩展解析事件并持续更新 UI（流式展示工具调用/进度/最终输出）。
- **没有“神秘 subagent API”**：这套做法本质是“用 CLI 模式把 pi 当作可被调用的 worker”，属于平台化的可组合能力，而不是 agent-core 内部的多代理特性。

## “pi 没有 subagent”到底指什么？
这里要区分“能力上做不到”与“产品层不内置”：

- `packages/agent`（agent-core）是一个单 agent runtime：它提供工具调用、消息状态、流式事件、`steer/followUp/continue` 等，但并没有一个“spawnSubagent()”之类的概念或协议。
- `packages/coding-agent`（产品 CLI/TUI）在理念上明确不把 sub-agents 做成内建功能，鼓励用 **扩展 / tmux / packages** 做你自己的多代理工作流。

## “自己调用自己”的两种含义（容易混淆）
你听到的“自己调用自己”，在 pi 生态里常见有两种解释：

### A) 进程内的“自续跑”（不是 subagent）
这是同一个 agent 对象在同一个上下文里继续执行，例如：
- `continue()`：从现有上下文继续（常用于错误后的重试、或处理队列消息）。
- `steer()/followUp()`：在 streaming 时把新的消息排队，让 agent 之后继续跑。

这类机制仍然是 **同一个 agent**，不产生新的上下文窗口。

### B) 进程外的“pi 调 pi”（subagent 扩展的核心）
这是把 pi 当作一个可调用的 worker 程序：在一个扩展/脚本里启动新的 `pi` 进程去跑任务，然后把输出再汇总回来。  
子进程的 system prompt、tools、model、是否落盘等都可以通过 CLI flags 控制。

这类机制会产生 **新的上下文窗口**（新进程 = 新 state），更接近你理解的 subagent。

## subagent 扩展示例的工作原理（按时间线，纯文字）
下面以 `subagent/` 扩展示例为“参考实现”，解释它如何完成“委派 → 子进程执行 → 回传结果”。

### 1) 定义“子代理人格/能力”来源：agents/*.md
扩展会从两个位置发现 agent 定义文件（Markdown + frontmatter）：
- 用户级：`~/.pi/agent/agents/*.md`
- 项目级：`.pi/agents/*.md`（可选，默认不启用或需要确认）

每个 agent 定义里通常包含：
- name/description（用于选择与展示）
- tools（限制子进程可用工具集合）
- model（指定子进程使用的模型）
- system prompt（写在 Markdown body 里）

### 2) 构造子进程 pi 的启动参数（关键点：模式与隔离）
扩展为每个子任务构造一组 CLI 参数，大意是：
- `--mode json`：让子进程输出 “JSON 事件流”
- `-p`：走非交互 print mode（单次执行、输出、退出）
- `--no-session`：不写 session 文件，子任务是“临时工”
- `--model/--tools`：从 agent 定义里带过去（让每个子代理各司其职）

扩展还会把 agent 的 system prompt 写到一个临时文件中，并通过 `--append-system-prompt <file>` 注入给子进程（从而做到“每个子代理不同人格/约束”）。

### 3) 启动子进程（pi 调 pi）并接管 stdout/stderr
扩展使用系统进程创建能力启动 `pi`：
- `stdout`：按行读取 JSON，解析成事件
- `stderr`：聚合为错误输出（作为失败原因之一）

### 4) 把 JSON 事件流转换成“可展示的进度与最终结果”
在 JSON 输出模式下，子进程会把 session 的事件逐条输出。

扩展会重点关注两类事件来构建“子任务的时间线”：
- assistant 消息结束（能拿到本轮输出与 usage）
- 工具结果结束（能把工具调用/结果作为可观测的执行轨迹展示）

扩展同时会把这些中间状态作为 tool 的 streaming update 回传给主 UI（所以你能在主界面看到 subagent 的实时进度，而不是只在最后拿结果）。

### 5) 终止/中断（Ctrl+C 传播到子进程）
扩展把 abort 信号绑定到子进程：
- 主会话 abort → 发送 SIGTERM（必要时升级 SIGKILL）杀掉子 pi 进程
- 子任务以 aborted 结束后，扩展把它作为失败向上汇报（避免静默挂起）

### 6) 编排模式（single / parallel / chain）
扩展示例并不止能跑单个子任务，它提供三种编排方式：
- single：一个 agent + 一个 task
- parallel：多个任务并行（有最大数量与最大并发度限制）
- chain：顺序执行，把上一步输出通过占位符注入下一步（实现 scout→plan→work 等流水线）

## 这种“subagent=子进程”的取舍
优点：
- **上下文隔离强**：子任务不会把主会话消息历史越写越长，且不会互相污染。
- **工具/模型可差异化**：不同 agent 可以使用不同 model、不同 tools（更符合“专精分工”的直觉）。
- **工程上简单可组合**：只要 CLI 稳定，这就是一个标准的 worker 模式；也方便把 subagent 放到远端/容器里做隔离。

代价：
- **进程开销**：启动新进程、加载资源、初始化模型等都要成本。
- **可观测性需要你做聚合**：多个子进程的事件流需要父扩展做汇总，否则会很乱。
- **安全边界更重**：项目级 agent 定义本质是“repo 控制的高权限指令文件”，示例扩展默认要求确认就是为了降低踩坑概率。

## 证据锚点（精选）
- `packages/coding-agent/README.md:410`：明确写了 “No sub-agents”，并建议用 tmux 或 extensions/third-party packages 实现你的方式。
- `packages/coding-agent/docs/extensions.md:1874`：官方扩展示例列表包含 `subagent/`（Spawn sub-agents）。
- `packages/coding-agent/examples/extensions/subagent/README.md:7`：说明隔离方式是“每个 subagent 在独立 `pi` 进程里运行”。
- `packages/coding-agent/examples/extensions/subagent/index.ts:1`：注释说明 spawn 独立 `pi` 进程，并用 JSON mode 捕获结构化输出。
- `packages/coding-agent/examples/extensions/subagent/index.ts:247`：子进程参数包含 `--mode json -p --no-session`（print+json+ephemeral）。
- `packages/coding-agent/examples/extensions/subagent/index.ts:280`：通过 `--append-system-prompt` 注入子代理的 system prompt（来自临时文件）。
- `packages/coding-agent/examples/extensions/subagent/index.ts:287`：用 `spawn("pi", args, ...)` 启动子进程并接管 stdout/stderr。
- `packages/coding-agent/examples/extensions/subagent/index.ts:347`：abort 时向子进程发送 SIGTERM/SIGKILL（Ctrl+C 传播）。
- `packages/coding-agent/src/cli/args.ts:10`：CLI `--mode` 支持 `json`（为事件流输出提供基础）。
- `packages/coding-agent/src/modes/print-mode.ts:75`：JSON 模式下会把 session 的所有事件逐条输出到 stdout。
- `packages/coding-agent/src/main.ts:366`：`--no-session` 会使用 in-memory session（不落盘）。
- `packages/agent/README.md:102`：`continue()` 是“从现有上下文继续”而不是 subagent（用于重试/恢复），避免把它误解为多代理。

## 边界/未验证点
- `subagent/` 位于 `examples/extensions/`：它是参考实现，不代表 core 承诺的长期稳定 API。
- 我没有在这次记录里把 subagent 的“JSON 事件集”逐条枚举成协议文档；这里只说明了它如何依赖 print-mode 的 JSON 输出。
- 如果你希望“同进程多 Agent 对象（多 session）”而不是多进程，也可以用 SDK 创建多个 `AgentSession`；但那是另一种实现路径，本笔记聚焦“自己调用自己（多进程）”。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)
- [extensions（扩展）机制：设计与实现](extensions-design-and-implementation.md)

