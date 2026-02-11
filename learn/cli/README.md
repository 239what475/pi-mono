# cli

这个目录用于记录与 `cli`（命令行入口与参数）相关的研究笔记。

范围：
- 关注各 CLI 的入口文件、子命令/参数解析、输出协议（例如 json/rpc）、打包方式。
- 不深入某个子命令的业务细节（放到 `agents` / `pods` / `llm` 等）。

关键词（用于归类/检索）：
- cli
- args
- commands
- json
- rpc
- binaries

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `packages/coding-agent/src/cli.ts` - coding agent CLI 入口
- `packages/coding-agent/src/main.ts` - 主流程入口（模式路由等）
- `packages/ai/src/cli.ts` - `@mariozechner/pi-ai` CLI 入口
- `packages/pods/src/cli.ts` - pods CLI 入口
- `scripts/build-binaries.sh` - 二进制构建脚本（如存在）
- `pi-test.sh` - 从源码运行 pi 的入口脚本

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## cli` 小节

