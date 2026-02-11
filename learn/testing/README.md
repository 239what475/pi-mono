# testing

这个目录用于记录与 `testing`（测试策略与测试工具链）相关的研究笔记。

范围：
- 关注各包的测试框架、测试入口脚本、需要凭证的测试（LLM 依赖）与跳过策略。
- 不记录具体功能点的断言细节（放到对应 topic 的笔记里）。

关键词（用于归类/检索）：
- vitest
- node:test
- fixtures
- e2e
- ci

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `test.sh` - repo 级测试入口脚本
- `packages/ai/vitest.config.ts` / `packages/ai/test/` - ai 测试与配置
- `packages/agent/vitest.config.ts` / `packages/agent/test/` - agent 测试与配置
- `packages/coding-agent/vitest.config.ts` / `packages/coding-agent/test/` - coding-agent 测试与 fixture
- `packages/tui/test/` - TUI 测试（Node 内置测试）

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## testing` 小节

