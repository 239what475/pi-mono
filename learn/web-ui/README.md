# web-ui

这个目录用于记录与 `web-ui`（浏览器端 chat UI 组件与存储）相关的研究笔记。

范围：
- 关注 web 端组件结构、消息/状态管理、存储（provider keys 等）、浏览器侧工具集成。
- 不深入 LLM provider 适配（放到 `llm`）或 agent loop（放到 `agents`）。

关键词（用于归类/检索）：
- web components
- chat
- storage
- prompts
- tools

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `packages/web-ui/README.md` - 使用方式与架构说明
- `packages/web-ui/src/index.ts` - 导出入口
- `packages/web-ui/src/ChatPanel.ts` - 核心 UI 容器（如适用）
- `packages/web-ui/src/components/` - UI 组件
- `packages/web-ui/src/storage/` - 本地存储（API keys、设置等）
- `packages/web-ui/example/` - 示例工程

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## web-ui` 小节

