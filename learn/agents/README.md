# agents

这个目录用于记录与 `agents`（agent runtime、工具调用、状态管理、coding agent 编排、集成）相关的研究笔记。

范围：
- 关注 agent loop、消息/事件模型、工具调用协议、coding agent 的模式（interactive/print/rpc）与扩展点。
- provider 适配与模型细节放到 `llm`；UI 细节放到 `tui` / `web-ui`。

关键词（用于归类/检索）：
- agent
- tool calling
- state
- session
- extensions
- modes

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `packages/agent/README.md` - agent core 总览
- `packages/agent/src/agent.ts` / `packages/agent/src/agent-loop.ts` - 核心运行逻辑
- `packages/coding-agent/README.md` - coding agent 使用与模式说明
- `packages/coding-agent/src/main.ts` - 主流程与模式路由
- `packages/coding-agent/src/core/` - session、tools、compaction、扩展等核心模块
- `packages/coding-agent/src/modes/` - interactive/print/rpc 等模式实现
- `packages/mom/src/slack.ts` - Slack 集成入口（如适用）

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## agents` 小节

