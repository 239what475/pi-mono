# config

这个目录用于记录与 `config`（配置来源与存储）相关的研究笔记。

范围：
- 关注配置的来源优先级（env、文件、默认值）、配置文件路径、持久化存储格式。
- 不记录具体 provider/模型列表（放到 `llm`）。

关键词（用于归类/检索）：
- config
- env
- settings
- storage
- schema

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `packages/ai/src/env-api-keys.ts` - 从环境变量解析 API key/凭证
- `packages/coding-agent/src/config.ts` - coding agent 配置入口
- `packages/pods/src/config.ts` - pods 配置入口
- `packages/mom/src/store.ts` - mom 侧的本地存储/状态（如适用）
- `packages/web-ui/src/storage/` - web-ui 的持久化（浏览器端）
- `packages/pods/docs/` - pods 配置/约定文档

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## config` 小节

