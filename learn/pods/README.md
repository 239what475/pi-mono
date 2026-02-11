# pods

这个目录用于记录与 `pods`（GPU pod 管理与 vLLM 部署）相关的研究笔记。

范围：
- 关注 pod 注册/切换、SSH/远程执行、模型部署参数、配置持久化与运维流程。
- 不记录统一 LLM API 的 provider 适配（放到 `llm`）。

关键词（用于归类/检索）：
- vllm
- gpu
- pod
- ssh
- deployment
- openai-compatible

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `packages/pods/README.md` - 功能概览与快速上手
- `packages/pods/src/cli.ts` - CLI 入口
- `packages/pods/src/commands/` - 子命令实现
- `packages/pods/src/ssh.ts` - SSH 相关封装
- `packages/pods/src/model-configs.ts` - 已知模型配置
- `packages/pods/scripts/` - pod setup / model run 脚本
- `packages/pods/docs/` - 部署/模型参数说明

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## pods` 小节

