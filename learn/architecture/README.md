# architecture

这个目录用于记录与 `architecture` 相关的研究笔记。

范围：
- 关注 pi-mono 的整体分层、包之间依赖边界、运行时/构建产物形态。
- 不记录具体 provider 或某个 UI 组件的实现细节（这些放到对应 topic）。

关键词（用于归类/检索）：
- monorepo
- workspaces
- packages
- layering
- entrypoints

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `README.md` - monorepo 总览与包列表
- `package.json` - workspaces + scripts（build/check/release）编排
- `packages/*/package.json` - 各包入口与依赖边界
- `tsconfig.base.json` / `tsconfig.json` - TypeScript 配置
- `scripts/` - release/version 同步脚本
- `packages/` - 各子包源码根目录

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## architecture` 小节

