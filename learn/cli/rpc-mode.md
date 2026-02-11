# cli：RPC mode（stdin/stdout JSON 协议）

日期：2026-02-11
主题：cli（learn/cli/）
结论一句话：RPC mode 是一种“常驻子进程 + JSON Lines 协议”，通过 stdin 接收命令（`prompt`/`abort`/`get_state`/…），通过 stdout 输出 `response`（命令是否受理/执行结果）与 `AgentSessionEvent`（message/turn/tool 生命周期事件流）；适用于把 pi 集成进 IDE、GUI、服务端守护进程或任何非 TUI 的宿主环境。

## TL;DR
- 启动：`pi --mode rpc [options]`（进程常驻，等待 stdin JSON 命令）。
- 输入（stdin）：一行一个 JSON object：`RpcCommand` 或 `extension_ui_response`。
- 输出（stdout）：一行一个 JSON object：`response`（命令结果）或事件（`AgentSessionEvent`），以及 extension UI 的 `extension_ui_request` / `extension_error`。
- `prompt` 是异步：RPC 会先返回 `response` 作为“已受理”，真正的生成/工具执行进度通过事件流输出；完成以 `agent_end` 为准。
- extensions 在 RPC mode 也能跑：通过 `extension_ui_request` 让宿主弹窗/输入；但很多 TUI 相关能力（主题切换、工具展开、working loader 等）在 RPC mode 会被禁用或降级。
- 如果你是 Node/TS 应用内集成：优先直接用 `AgentSession`（不需要子进程）；RPC 更适合“跨语言/跨进程”集成。

## 为什么需要 RPC mode（它解决什么问题）
- **把交互模式拆成协议**：TUI/GUI/IDE 不需要复刻 pi 的内部逻辑，只要实现“发送命令 + 消费事件流”即可。
- **允许宿主掌控 UI/权限/生命周期**：宿主可以决定如何展示 streaming、如何确认危险操作、如何呈现 extension UI。
- **同一套核心事件流可复用**：RPC 输出的是 `AgentSessionEvent`，与 interactive/json mode 的事件语义一致（message/tool/turn/agent 生命周期）。

## 协议结构（你需要实现的最小集合）

### 1) Commands（stdin）
- 一行一个 JSON object，核心字段是 `type`（可选 `id` 用于关联 response）。
- 典型命令：`prompt`、`get_state`、`abort`、`bash`、`compact`、`set_model` 等。

### 2) Responses（stdout）
- `type: "response"`，包含：
  - `command`：对应命令类型
  - `success`：true/false
  - `data`：可选（例如 `get_state`/`bash`/`compact` 等会返回数据）
  - `error`：失败时的错误信息

### 3) Events（stdout）
- 所有 `AgentSessionEvent` 都会被逐行输出（例如 `message_update` 的 text delta、`tool_execution_*` 的工具进度等）。
- 一般做法：宿主把 stdout 的 JSON line 分流：
  - `response` → 解决 pending request
  - 其他 → 作为事件交给 UI 渲染/状态机

### 4) Extension UI（stdout + stdin）
- extension 如果需要“像 UI 一样问用户问题”，RPC mode 会输出 `extension_ui_request`（带 `id`）。
- 宿主收到后自己弹窗/输入，然后把对应 `extension_ui_response` 写回 stdin（必须带同一个 `id`）。
- 只有对话框类方法需要 response；`notify`/`setStatus`/`setWidget`/`setTitle`/`set_editor_text` 这类是 fire-and-forget。

## 一步一步：宿主应该怎么接入（推荐实现顺序）

### Step 1：启动子进程
启动方式（概念上）：
1) `spawn("pi", ["--mode", "rpc", ...options])` 或 `node dist/cli.js --mode rpc ...`
2) stdout 按行读取、逐行 JSON.parse
3) stdin 写入 JSON.stringify(command) + `\n`

### Step 2：先实现 request/response 关联（`id`）
把每个 outgoing command 带上 `id`；收到 stdout：
- 若是 `{ type: "response", id, ... }`：resolve/reject 对应请求
- 否则：当作事件派发给渲染层/状态机

### Step 3：最小可用的 prompt streaming
发送：
```json
{"type":"prompt","id":"req-1","message":"Hello"}
```
宿主应该：
- 立刻收到 `response`（ack）
- 之后持续收到事件（`agent_start`、`message_update` 的 delta、`tool_execution_*`、最终 `agent_end`）
- “一次 prompt 完成”的判定：看到 `agent_end`

### Step 4：处理 streaming 中的用户新输入（steer/follow-up）
当宿主在 streaming 中又要发新指令时：
- 用 `prompt` 并附带 `streamingBehavior`（`steer` 或 `followUp`）
- 或直接用 `steer` / `follow_up` 命令

### Step 5：支持 extensions 的 UI 交互（可选但非常关键）
收到：
```json
{"type":"extension_ui_request","id":"uuid-1","method":"select", ...}
```
宿主弹窗后回写：
```json
{"type":"extension_ui_response","id":"uuid-1","value":"Allow"}
```

### Step 6：补齐状态/运维命令（按需）
- `get_state`：驱动 UI 的 footer、状态栏、连接状态等
- `abort` / `abort_bash` / `abort_retry`：给宿主“停止按钮”
- `get_messages`：用于回放、导出、或宿主侧的消息列表
- `export_html`：把 session 导出为 HTML（宿主可以直接打开/分享）

## 与 JSON mode 的区别（避免混用）
- JSON mode：`pi --mode json "prompt"` → **一次性运行**，把事件打印出来后结束；适合 pipeline/脚本。
- RPC mode：`pi --mode rpc` → **常驻进程**，通过 stdin 收多条命令；适合 IDE/GUI/服务集成。

## 限制/边界（RPC mode 的“没有”）
- stdin 被协议占用：不会读取 piped stdin 作为输入消息；也不支持 `@file` CLI 参数（文件内容应由宿主自己读取后放进 `prompt.message`）。
- 没有 TUI：主题切换、工具输出展开/折叠、working loader、custom header/footer、custom editor component 等都会被禁用或降级；extensions 若依赖这些能力，需要在宿主侧实现或在扩展中做降级处理。

## 证据锚点（精选）
- `packages/coding-agent/docs/rpc.md:1`：RPC mode 定义为 stdin/stdout 的 JSON 协议，并给出命令/事件/扩展 UI 的说明。
- `packages/coding-agent/src/main.ts:587`：RPC mode 会跳过 piped stdin 读取（stdin 留给 RPC 协议）。
- `packages/coding-agent/src/main.ts:612`：RPC mode 禁止 `@file` 参数（因为 stdin/文件输入策略不同）。
- `packages/coding-agent/src/main.ts:690`：`mode === "rpc"` 时进入 `runRpcMode(session)`。
- `packages/coding-agent/src/modes/rpc/rpc-mode.ts:45`：`runRpcMode()` 说明：stdin 收命令、stdout 输出 events + responses。
- `packages/coding-agent/src/modes/rpc/rpc-mode.ts:120`：创建 RPC 版 `ExtensionUIContext`（把扩展 UI 交互转成 `extension_ui_request/response`）。
- `packages/coding-agent/src/modes/rpc/rpc-mode.ts:271`：`session.bindExtensions(...)`：在 RPC mode 下仍启用 extensions，并把 UI/command actions 映射为 RPC 能力。
- `packages/coding-agent/src/modes/rpc/rpc-mode.ts:310`：`session.subscribe(event => output(event))`：把所有 session 事件逐行输出到 stdout。
- `packages/coding-agent/src/modes/rpc/rpc-mode.ts:324`：`prompt` 命令：不 await，事件异步流式输出。
- `packages/coding-agent/src/modes/rpc/rpc-mode.ts:598`：readline 按行读取 stdin JSON；分流 `extension_ui_response` 与普通命令。
- `packages/coding-agent/src/modes/rpc/rpc-types.ts:18`：`RpcCommand`/`RpcResponse`/`RpcExtensionUIRequest`/`RpcExtensionUIResponse` 类型定义。
- `packages/coding-agent/src/modes/rpc/rpc-client.ts:54`：提供一个“typed RPC client”示例：spawn + 按行解析 stdout + 通过 id 关联 response。
- `packages/coding-agent/test/rpc.test.ts:14`：RPC mode 的集成测试（需要 API key），覆盖 get_state、prompt 持久化、compaction、bash 等。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)

## 后续问题
- 宿主侧如何设计“事件驱动 UI 状态机”（例如把 `message_update` 的 delta 合并成可回放的消息结构）。
- extensions 的 UI 请求（select/confirm/input/editor）在宿主侧如何与自己的 modal 系统对接。

