# agents：pi coding-agent 的 extensions（扩展）机制：设计与实现

日期：2026-02-11  
主题：agents（learn/agents/）  
结论一句话：`packages/coding-agent` 的 extensions 是“宿主加载并执行的 TypeScript 模块”，通过注册（tools/commands/shortcuts/flags/renderers/handlers/providers）+ 运行期绑定（UI/Session/Agent actions）实现可插拔能力；LLM 只能调用已注册的工具，不能在运行时凭空“写出并启用”新工具。

## TL;DR
- **两阶段**：加载期（extension factory 只做“注册”）+ 运行期（`ExtensionRunner` 绑定 session/agent/ui 后才可“执行动作/跑工具/发消息”）。
- **扩展不是提示词魔法**：extension 本质是本地代码，具备完整宿主权限（可执行命令、读写文件、发 UI 请求等），因此是**信任边界**。
- **工具接入方式**：extension 注册 `ToolDefinition` → 被包装成 `AgentTool`（注入 `ExtensionContext`）→ 再被“二次包装”以发出 `tool_call/tool_result` 事件（可阻止/可改写结果）。
- **事件模型是主骨架**：`ExtensionRunner.emit*()` 顺序执行所有 extension 的 handler；大多数事件“失败不炸主流程”，通过 `onError` 上报。
- **UI 通过 context 注入**：mode（interactive/rpc/print）负责提供 `ExtensionUIContext`；无 UI 时 runner 提供 no-op UI，extension 仍可跑但 UI 操作会失败/被忽略。
- **资源可被扩展**：`resources_discover` 事件允许 extension 动态追加 skill/prompt/theme 路径，`AgentSession` 会把这些路径灌入 `ResourceLoader` 并重建 system prompt。

## 核心心智模型：extension 是“可插拔的宿主代码层”
把 extension 机制理解为一套“把第三方代码接入 pi runtime 的插槽系统”会更准确：

- **它不是 LLM 的能力**：LLM 只能在“宿主已经把工具/命令暴露出来”的前提下做 tool calling。
- **它是宿主的扩展点**：extension 运行在本地进程内，能读写文件、执行命令、发 UI 请求、影响 session 与 system prompt。
- **它主要解决两类需求**：
  1) 提供新的能力：新工具、新命令、快捷键、主题、skills、prompt templates  
  2) 统一治理：拦截所有工具调用、审计/记录、内容脱敏、输入改写、策略阻断、自动插入上下文等

## 如何启用 extension（放哪、怎么被发现）
你可以把 extension 理解为“会被 pi 启动时加载的一段本地 TS/JS 代码”。常见的启用方式有三类（按优先级/易用性理解，不代表严格顺序）：

1) **用户级（全局）扩展目录**  
- 默认：`~/.pi/agent/extensions/`  
- 适用：你希望所有项目都能用的工具（比如公司内网 proxy、统一的审计工具、个人工作流命令）。

2) **项目级扩展目录**  
- 默认：`<project>/.pi/extensions/`  
- 适用：只对当前 repo 生效的工具（例如仓库特定的脚手架、内部 API 调用工具、项目规范检查等）。

3) **settings.json 显式配置路径**  
- 你可以在 settings 里写 `extensions: ["./some/dir-or-file", "/abs/path/to/ext.ts"]`，支持“文件或目录”。  
- 目录的发现规则是“单层”：
  - `extensions/*.ts|*.js` 直接作为入口
  - `extensions/<dir>/index.ts|index.js` 作为入口
  - 或在 `extensions/<dir>/package.json` 里用 `pi.extensions` 明确声明入口（支持多个入口文件）

说明：
- `CONFIG_DIR_NAME` 默认是 `.pi`，但可被 `package.json` 的 `piConfig.configDir` 覆盖；用户级目录里的 `agent` 也可以被环境变量 `${APP_NAME}_CODING_AGENT_DIR`（例如 `PI_CODING_AGENT_DIR`）重定向。
- 复杂/多文件的扩展推荐用 `package.json` + `pi.extensions` 管理入口，避免依赖扫描规则。

## 运行时对象与职责（先认清“谁负责什么”）
下面这些对象名在实现里会反复出现，你读源码/调试时建议先把它们对上号：

- **Extension（声明集合）**：每个扩展模块对应一个 `Extension`，内部是若干 Map（handlers/tools/commands/flags/shortcuts/renderers）。
- **ExtensionAPI（扩展作者看到的 pi 对象）**：一边提供“注册方法”（写入 Extension 的 Map），一边提供“动作方法”（委派给 runtime，在运行期才会真正可用）。
- **ExtensionRuntime（共享动作后端）**：所有 extensions 共享一个 runtime；加载期它是“禁止动作”的占位实现；运行期由 runner/core 把真实动作绑定进去。
- **ExtensionRunner（调度器）**：持有所有 extensions + runtime + 当前 UI/session/model 的绑定关系；统一负责：
  - 提供 `createContext()` 生成 ExtensionContext
  - 发事件（emit/emitToolCall/emitToolResult/emitInput…）
  - 收集已注册的 tools/commands/shortcuts/flags/renderers
  - 将异常通过 onError 向上汇报
- **ExtensionContext（每次执行/回调的运行态上下文）**：extension 在 handler/tool/command 里真正使用的“宿主能力入口”，包括 UI、cwd、sessionManager、modelRegistry、abort/compact/getSystemPrompt 等。
- **ExtensionUIContext（UI 抽象层）**：interactive 模式会连到 TUI；rpc 模式会转成“发请求/收响应”的远程 UI；无 UI 时是 no-op（不会崩，但多数 UI 操作不产生效果）。

## 设计拆解：加载期 vs 运行期

### 1) 加载期（extension “被发现/被 import/被初始化”）
目标：把 extension 的“声明”收集起来（工具、命令、事件 handler 等），但**禁止**它在加载阶段对 session 做动作。

加载期发生的事：
1. 为所有 extensions 创建一个共享的 `ExtensionRuntime`，其中 action 方法是“抛错的占位实现”（防止加载期调用）。  
2. 每个 extension 都有自己的 `Extension` 对象（Map 集合），`ExtensionAPI` 的注册方法把数据写进这些 Map。
3. 通过 `jiti` 动态 import extension 模块，要求其 `default export` 是一个 factory 函数；执行 factory 时传入 `ExtensionAPI`，factory 内调用 `pi.registerTool/on/registerCommand/...` 完成注册。

你可以把加载期理解为：
> “收集插件的声明（registration）”，不要在这里做任何依赖运行态的事情。

### 2) 运行期绑定（把 extension 接到 session/agent/ui 上）
目标：把“加载期收集的声明”接入实际运行环境，让 extension 能：
- 发消息（custom / user）
- 调度 session（切换/分叉/导航树/重载）
- 得到当前 model、context usage、是否 idle 等运行态信息
- 使用 UI（选择/确认/输入/状态栏/编辑器等）

关键点：
- `AgentSession._buildRuntime()` 在创建/重建运行时，会创建 `ExtensionRunner`，把它当作“extension 的执行中枢”。  
- `AgentSession._bindExtensionCore()` 把 action（sendMessage、setActiveTools、setModel、compact、shutdown…）绑定进 runner 的 runtime。  
- mode 再通过 `AgentSession.bindExtensions()` 注入 `uiContext`、command-only actions、shutdown handler、error listener。

### 3) 运行期执行（事件 + 工具执行）
extension 的代码主要在两类场景运行：
1) **事件 handler**：例如 `input`、`before_agent_start`、`context`、`tool_call`、`tool_result`、各种 session/turn 事件。  
2) **工具 execute**：由 LLM 发起 tool call，宿主实际执行 tool（其中 extension 工具通过 ctx 获取 UI/session/model 等）。

## 生命周期时间线（从启动到“一次 prompt 完成”）
这一段用“时间顺序”讲清楚 extension 机制到底在什么时候介入主流程。

### 阶段 A：启动/加载（只做注册）
1. **收集 extension 路径**：来自用户级目录、项目级目录、settings 显式配置路径，以及可能的 CLI 参数/安装包资源（具体来源由 resource loader + package manager 合并）。
2. **逐个加载模块**：每个 extension 都会被动态加载，拿到 default export 的 factory。
3. **执行 factory 完成注册**：factory 只应该调用“注册方法”（注册工具/命令/快捷键/flag/renderer/handler…）。  
   - 如果 factory 试图在加载期调用动作方法，会因为 runtime 未初始化而失败（这是设计使然：避免“初始化即开跑”的副作用）。

### 阶段 B：创建运行时（把扩展插到 session/agent 上）
4. **构建工具与系统提示词基线**：coding-agent 会先构建内置工具集合与基础 system prompt（包含 skills、context files、选中的工具等）。
5. **创建 ExtensionRunner 并绑定 core actions**：如果存在扩展或 SDK 自定义工具，则创建 runner，并把“真实动作”绑定进去（例如发消息、切换工具、选择模型、compact、shutdown 等）。
6. **mode 注入 UI/命令上下文**：interactive/rpc/print 等模式，决定 extension 的 UI 能力是“真 UI”“远程 UI”还是“无 UI”，同时补齐那些只允许用户发起的 session 控制动作（newSession/fork/switch/reload 等）。

### 阶段 C：会话启动（让扩展有机会补资源）
7. **session_start 事件**：当 runner 已经绑定完成，宿主会发 session_start（这是扩展“可以开始做运行期事情”的信号）。
8. **resources_discover 事件（可选）**：若扩展提供了该 handler，就可以返回额外的 skill/prompt/theme 路径；宿主会把这些路径合并进资源系统，然后重建 system prompt，让新增资源在当前 session 立即生效。

### 阶段 D：用户输入与 prompt（扩展可拦截/改写/抢先处理）
9. **extension command（以“/”开头的命令）优先执行**：当用户输入以 `/` 开头时，宿主会先尝试把它当作“扩展命令”执行；如果命中，会立刻执行命令 handler 并结束本次 prompt（不再进入 LLM turn）。  
   - 这个设计使“命令”成为一条独立的控制通道：可在 streaming 时立即执行，不需要排队。
10. **input 事件（在 skills/prompt template 展开前）**：如果扩展注册了 input handler，宿主会在展开 `/skill:` 与 `/template` 之前先发 input 事件。  
    - 扩展可以选择：继续、把输入改写成新文本/新图片、或直接标记 handled（吞掉输入）。
11. **skills 与 prompt templates 展开**：input 事件之后，宿主才会把 `/skill:name` 和 `/template` 这类文本扩展成真正发给模型的内容。
12. **before_agent_start（最后一次“改写上下文”的机会）**：在真正调用 agent 前，扩展可以：
    - 注入若干“custom messages”作为同一轮的上下文（通常用于补充背景、记录、指令片段）
    - 临时改写 system prompt（只针对这次 turn；下一次会恢复基线）
13. **agent loop 开始**：接下来就进入 agent-core + ai provider 的流式调用；扩展不再直接参与“文本生成”，但可以通过 tool 机制参与执行。

### 阶段 E：工具调用（扩展既能提供工具，也能拦截所有工具）
14. **模型提出 tool call**：LLM 在流中发出“我要调用工具 X（参数 Y）”的意图。
15. **宿主执行工具（统一被扩展拦截层包裹）**：在真正执行工具前会发 `tool_call` 事件；扩展可以阻止；执行后会发 `tool_result` 事件；扩展可以改写结果。
16. **工具结果回流到 agent**：工具结果写入 session，也作为 tool result 回到 agent loop，继续生成后续文本或触发更多工具调用，直到 turn 结束。

## 工具（tools）是怎么“被写出来并被 LLM 调用”的？

### A. 注册：ToolDefinition
extension 通过 `pi.registerTool({ name, description, parameters, execute, ... })` 注册工具。

ToolDefinition 的关键点：
- `execute(..., ctx: ExtensionContext)`：每次执行都会拿到运行态 ctx（UI + session + model + utilities）。
- `onUpdate`：允许 tool 在执行中**流式更新**（例如长任务、进度条、分段输出）。
- `renderCall/renderResult`：可选的 UI 渲染器（TUI/GUI 层会用它决定如何展示）。

### B. 变成 AgentTool：wrapRegisteredTool()
runner 会把 `ToolDefinition` 包装成 `AgentTool`，并在调用 `execute` 时注入 `runner.createContext()`。

### C. 被“扩展回调”包裹：wrapToolWithExtensions()
无论是内置工具还是 extension 工具，最终都会被再包一层：
- 执行前发 `tool_call`（可 `block` 阻止执行）
- 执行后发 `tool_result`（可修改 content/details/isError）

这意味着 extension 既可以“提供新工具”，也可以“对所有工具加拦截/审计/脱敏/策略控制”。

## 事件系统：ExtensionRunner 是“统一调度器”
`ExtensionRunner` 的职责是：
- 维护 extensions 列表与它们注册的 handlers/tools/commands/shortcuts/flags/renderers
- 提供 `emitXxx()` 方法把事件按顺序派发给所有 extension
- 捕获错误并上报（不会让单个 extension 直接炸掉主流程）

几个“事件语义不同”的例子：
- `input`：transform chain；遇到 `{action:"handled"}` 会短路停止后续 handler。  
- `context`：允许把 messages 数组改写（链式传递）。  
- `tool_call`：允许 block（类似前置策略）。  
- `tool_result`：允许修改输出（链式 patch）。  
- `session_before_*`：可 cancel（拦截 session 切换/分叉/compact/tree）。

## 命令/快捷键/渲染器/flags/provider：除了工具以外还能扩什么？
为了避免你把 “extension = tools” 理解得太窄，这里把其它注册能力的定位讲清楚：

- **commands（斜杠命令）**：用户输入 `/xxx args` 时触发。它和“发给模型的 prompt”是两条不同路径：命令是控制面，适合做会话管理、工作流指令、批处理任务、把复杂交互封装成可重复操作。  
  另外：宿主会过滤与内置命令同名的扩展命令（避免抢占关键内建功能）。
- **shortcuts（快捷键）**：扩展可以注册按键，但宿主对一部分关键动作做了“不可覆盖”的保留（避免扩展抢掉退出/中断等关键操作）；非保留动作允许覆盖但会提示冲突。
- **message renderers（渲染器）**：扩展可以为自定义消息类型提供专用 UI 渲染方式，让输出不只是纯文本（例如进度、表格、结构化结果）。
- **flags（命令行标志）**：扩展可声明 flag，并在运行时读取其值；适合把扩展行为做成可配置开关或参数。
- **provider 注册**：扩展可以注册/覆盖模型 provider（例如走公司代理、加 OAuth、自定义 stream handler），但这类注册在加载期通常会被“延后”到运行期绑定后才真正落到 model registry 上。

## 资源扩展：把 extension 带来的 skills/prompts/themes 注入系统
extension 可以监听 `resources_discover` 并返回路径列表：
- skills：额外的 `SKILL.md`
- prompts：额外的 prompt templates
- themes：额外的主题文件

`AgentSession` 在 `session_start` 后会触发 discover，并把这些路径作为“temporary scope 的资源”扩展进 `ResourceLoader`，然后重建 system prompt（确保 prompt templates/skills 的 slash command 列表等在当前 session 生效）。

## 安全边界（重要）
- extension 是**本地执行代码**：可以 `pi.exec()`，也能通过各种 handler 在用户输入、工具调用前后执行逻辑。
- 因此 “让 LLM 自己写工具” 的正确理解是：  
  1) LLM 可以生成 extension 代码（文本）  
  2) 但必须由你把它保存到扩展目录/配置路径，并允许 pi 加载  
  3) 加载后 LLM 才能通过 tool calling 调用它  
- 若你打算把 extension 当成“可安装插件生态”，需要额外的安全治理：来源、签名、沙箱、权限模型、review 流程等（当前实现默认假设 extension 是可信的）。

## 常见误解（快速澄清）
- “extension 是不是类似 prompt injection？”  
  不是。extension 是本地代码层；它能影响 prompt/上下文，但它本身不是 prompt。
- “agent 能不能自己写工具？”  
  LLM 可以生成 extension 源码文本，但启用它仍然需要宿主把它放到可加载路径并允许加载；这一步是信任与权限边界。
- “扩展会不会把 pi 的主流程搞崩？”  
  大多数事件 handler 的错误会被捕获并上报，不会直接让主流程崩；但拦截型点位（比如 tool_call 阻断）天然更偏 fail-closed，需要扩展作者自觉降低误伤。

## 证据锚点（精选）
- `packages/coding-agent/src/core/extensions/loader.ts:107`（`createExtensionRuntime()`）：加载期 runtime action 是抛错占位；防止扩展在初始化时就对 session/agent 产生副作用。
- `packages/coding-agent/src/core/extensions/loader.ts:136`（`createExtensionAPI()`）：注册方法写入 extension 声明集合；动作方法委派给共享 runtime（运行期绑定后才可用）。
- `packages/coding-agent/src/core/extensions/loader.ts:470`（`discoverAndLoadExtensions()`）：标准发现位置（全局/项目/显式配置路径）与单层 discovery 规则。
- `packages/coding-agent/src/core/extensions/runner.ts:235`（`bindCore()`）：把宿主动作绑定进 runtime，并处理加载期排队的 provider 注册。
- `packages/coding-agent/src/core/extensions/runner.ts:522`（`emit()`）：事件按扩展顺序派发；异常捕获并通过 `emitError()` 上报；部分事件支持 cancel。
- `packages/coding-agent/src/core/extensions/wrapper.ts:13`（`wrapRegisteredTool()`）：扩展工具执行时注入 `ExtensionContext`（让 tool 能访问 UI/session/model 等运行态能力）。
- `packages/coding-agent/src/core/extensions/wrapper.ts:38`（`wrapToolWithExtensions()`）：所有工具执行前后都会触发 `tool_call/tool_result`（可阻止、可改写）。
- `packages/coding-agent/src/core/agent-session.ts:654`（`prompt()`）：命令优先 → input 事件 → skills/template 展开 → streaming 队列逻辑 → 进入 agent。
- `packages/coding-agent/src/core/agent-session.ts:798`（`_tryExecuteExtensionCommand()`）：扩展命令解析与执行（命中则本次 prompt 被命令消费）。
- `packages/coding-agent/src/core/agent-session.ts:762`（`before_agent_start`）：扩展可注入 custom messages，并可临时改写 system prompt（本次 turn 生效）。
- `packages/coding-agent/src/core/agent-session.ts:1909`（`_buildRuntime()`）：把扩展工具与内置工具合并，并对 active/all tools 应用扩展拦截包装。
- `packages/coding-agent/src/core/agent-session.ts:1736`（`extendResourcesFromExtensions()`）：`resources_discover` 的返回路径会扩展到资源系统，并触发 system prompt 重建。

## 反例/边界情况
- extension 加载期不能调用 runtime action（会抛错）；如果你需要“启动时做一次动作”，应监听 `session_start` 或 `resources_discover`。
- discovery 规则是“单层”模型：复杂目录结构需要用 `package.json` 的 `pi.extensions` 指定入口（否则不会递归扫描深层）。
- 无 UI 或 RPC 模式下，某些 UI 能力不可用（例如工作消息、TUI widget 等），需要 extension 自行做降级处理。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)
- [一次 prompt 的完整事件链（输入→LLM→工具→UI）](prompt-to-ui-event-chain.md)
- [工具（tools）是谁写的？agent 能不能“自己写工具”】【边界澄清】](tools-authoring.md)
- [cli：RPC mode（stdin/stdout JSON 协议）](../cli/rpc-mode.md)

## 后续问题
- interactive mode 是如何把 `registerCommand/registerShortcut/registerMessageRenderer` 接到 TUI 渲染与按键分发上的？（下一步可从 `packages/coding-agent/src/modes/interactive/` 追）
