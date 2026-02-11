# security

这个目录用于记录与 `security`（凭证、权限、数据落盘、沙箱）相关的研究笔记。

范围：
- 关注 API keys / OAuth token 的来源与存储、日志/导出是否会泄露敏感信息、远程执行的边界。
- 不记录纯业务逻辑（放到对应 topic）。

关键词（用于归类/检索）：
- api keys
- oauth
- secrets
- storage
- sandbox

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `packages/ai/src/env-api-keys.ts` - 环境变量凭证解析
- `packages/web-ui/src/storage/` - 浏览器侧 key 存储策略（如适用）
- `packages/coding-agent/src/core/` - session/log/export 等可能涉及敏感信息的模块
- `packages/pods/src/` - 远程 SSH / vLLM API key 使用与传递
- `AGENTS.md` - 开发与工具使用约束（含安全相关约定）

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## security` 小节

