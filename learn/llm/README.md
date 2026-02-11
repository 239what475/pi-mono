# llm

这个目录用于记录与 `llm`（模型、provider、流式输出、工具调用、token/usage）相关的研究笔记。

范围：
- 关注统一 API 的类型设计、provider 适配层、模型发现/生成、stream 事件协议。
- 不记录上层 agent 的业务编排（放到 `agents`）。

关键词（用于归类/检索）：
- provider
- model
- streaming
- tool-calling
- tokens
- usage

入口线索（先列出最可能相关的文件/目录/符号，后续可修正）：
- `packages/ai/README.md` - 总览、使用方式、provider 列表
- `packages/ai/src/types.ts` - API/模型/选项等核心类型
- `packages/ai/src/stream.ts` - 统一 stream 入口与路由
- `packages/ai/src/providers/` - 各 provider 实现
- `packages/ai/src/models.generated.ts` - 生成的模型清单
- `packages/ai/scripts/generate-models.ts` - 模型发现/生成脚本
- `packages/ai/test/` - streaming、tokens、tool 相关测试

约定：
- 新笔记文件名用 kebab-case：`foo-bar.md`
- 把笔记链接登记到 `learn/SUMMARY.md` 的 `## llm` 小节

