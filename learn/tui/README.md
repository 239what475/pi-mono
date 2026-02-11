# tui

这个目录用于记录与 `tui`（终端 UI 与交互）相关的研究笔记。

范围：
- 关注终端渲染、输入处理、组件模型、键位/快捷键体系等。
- 不记录 agent loop 与模型/工具协议（放到 `agents` / `llm`）。

关键词（用于归类/检索）：
- terminal
- rendering
- keybindings
- components
- input

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `packages/tui/README.md` - TUI 库概览
- `packages/tui/src/index.ts` - 导出入口
- `packages/tui/src/keybindings.ts` / `packages/tui/src/keys.ts` - 键位定义与绑定
- `packages/tui/src/components/` - 组件实现
- `packages/tui/src/editor-component.ts` - 编辑器组件（如适用）
- `packages/tui/test/` - 单元测试与交互行为验证

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## tui` 小节

