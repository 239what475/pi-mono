# workflow

这个目录用于记录与 `workflow` 相关的研究笔记。

范围：
- 关注贡献流程、开发/发布脚本、常用命令、约束与约定。
- 不展开具体功能实现（放到对应 topic，例如 `llm` / `agents` / `pods`）。

关键词（用于归类/检索）：
- contributing
- build
- check
- release
- changelog
- versioning

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `CONTRIBUTING.md` - 贡献指南
- `AGENTS.md` - 项目内的开发/工具约束（对人和 agent）
- `package.json` - `scripts.build` / `scripts.check` / `scripts.release:*`
- `scripts/release.mjs` - 发布脚本
- `scripts/sync-versions.js` - monorepo 版本同步
- `pi-test.sh` / `test.sh` - 本地运行与测试入口
- `packages/*/CHANGELOG.md` - 各包变更记录（lockstep 版本）

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## workflow` 小节

