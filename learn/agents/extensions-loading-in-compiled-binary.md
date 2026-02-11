# agents：为什么“编译后的 pi”仍然能加载 extensions 代码？

日期：2026-02-11  
主题：agents（learn/agents/）  
结论一句话：pi 的 “compiled/打包” 并不等于“禁止动态执行代码”；`packages/coding-agent` 内置了一个运行时扩展加载器（jiti），可以在运行时从文件系统动态导入 TS/JS 扩展；在 Bun 编译成单文件二进制时，还通过 `virtualModules` 把必要依赖“内嵌进二进制并暴露给扩展”。

## TL;DR
- “编译过了还能加载代码”主要来自两点：**JS 运行时天然支持动态导入**，以及 pi **主动实现了 extension loader**。
- pi 的 extension 是“外部模块”：它不在主程序构建产物里，而是**运行时从磁盘加载**（所以可以更新/替换扩展而不重编译主程序）。
- 扩展入口文件可以是 `.ts`：因为 loader 使用 `jiti` 做 **运行时转译/加载**（而不是要求你先把扩展编译成 `.js`）。
- 在 **Bun 编译成二进制** 的场景下，主程序的依赖可能不再以 `node_modules` 形式存在；pi 用 `virtualModules` 解决“扩展 import 依赖”的问题。
- 因为这是动态代码执行，所以 extension 天然是**信任边界**：你加载什么，就等于允许它在本地进程里执行什么。

## 先澄清：你说的“编译过”可能是三种不同含义
很多人听到“编译过了”会联想到 C/C++ 的静态链接、不可再加载脚本。但在 JS/TS 生态里，“编译/打包”往往是下面几种之一：

1) **TypeScript → JavaScript（转译）**  
这不是把程序变成不可扩展的原生机器码，而是把 TS 变成 JS；JS 运行在 Node/Bun 里，天生就支持 `import()`/动态加载模块。

2) **bundle 成单文件/少文件产物**  
bundle 只是“把一堆依赖打包到一起”，不代表运行时不能再加载其它文件。是否允许动态加载，取决于运行时能力与程序是否实现了 loader。

3) **Bun 把 JS 打成单个可执行二进制（compiled binary）**  
这看起来最“像原生程序”，但本质仍然包含 Bun 的 JS runtime；程序仍然可以在运行时加载外部模块（前提是你写了加载器，并解决依赖解析问题）。

pi 的 extension 机制，正好覆盖了 (1)(2)(3)：无论你是通过 npm 运行、还是用 Bun 二进制运行，宿主都能在运行时加载扩展。

## pi 是怎么做到“运行时加载 TS/JS 扩展”的？
把机制拆成两层会更清晰：

### 层 1：运行时动态导入扩展入口文件
pi 在 `packages/coding-agent` 内实现了一个 extension loader：
- 它会把扩展入口路径解析成绝对路径（包含 `~` 展开等），然后在运行时对这个路径执行“导入模块”。
- “导入”不是 Node 的原生 `import()`，而是交给 `jiti`：这样它能直接处理 `.ts` 扩展入口（相当于运行时转译/执行）。

直观理解：  
> 主程序编译好了没关系，它只是“带着一个 JS 运行时和一个 loader”。loader 每次都可以去磁盘上找扩展入口文件并执行。

### 层 2：扩展在 Bun 二进制里如何 import 宿主依赖？
当你用 Bun 把程序做成“单个二进制”时，经常会出现一个现实问题：
- 二进制里不一定存在可供模块解析的 `node_modules` 目录结构（很多依赖被打进二进制内部了）。
- 但扩展想 import 的东西（例如 pi 提供的 SDK/types、TypeBox、pi-ai、pi-agent-core、pi-tui）又必须能解析到。

pi 的做法是：
- 把“扩展可能需要 import 的关键包”用 **静态 import** 的方式引入主程序（这样 Bun bundler 才会把它们打进二进制）。
- 在创建 `jiti` 时，如果检测到是 Bun binary，就启用 `virtualModules`：把这些内嵌模块映射成“可被扩展 import 的虚拟模块名”。

效果：即便没有磁盘上的 `node_modules`，扩展也能 import 到宿主预置的这些包。

## 这意味着什么（能力与限制）
能力：
- 你可以在不重编译/不重发布主程序的情况下，新增/替换 extension 文件，就能改变工具/命令/拦截逻辑等行为。
- 扩展可以用 TS 写，因为 loader 负责运行时处理。
- Bun 二进制分发仍能支持扩展生态，因为宿主把关键依赖“内嵌并虚拟化”给扩展用。

限制/边界：
- 扩展能 import 到哪些包，取决于运行时解析能力：
  - npm/Node 开发环境：通常靠 `node_modules` + alias 解析（较自由）
  - Bun binary：宿主显式提供的 `virtualModules` 能力更关键；扩展若依赖其它第三方包，通常需要自带依赖（例如扩展目录自身的 `node_modules`）或避免使用
- 动态加载意味着安全模型是“信任你加载的代码”。在不可信环境里应禁用扩展或引入额外隔离机制（沙箱/权限/签名/审核）。

## 证据锚点（精选）
- `packages/coding-agent/src/core/extensions/loader.ts:1`：loader 的注释明确说明使用 `jiti` 加载 TS 扩展，并支持“compiled Bun binaries”。
- `packages/coding-agent/src/core/extensions/loader.ts:17`：说明为什么要用静态 import（为了让 Bun 把依赖打进二进制），并通过 `virtualModules` 暴露给扩展。
- `packages/coding-agent/src/core/extensions/loader.ts:40`：`VIRTUAL_MODULES` 的注释与映射，证明“虚拟模块”机制的存在与意图。
- `packages/coding-agent/src/core/extensions/loader.ts:52`：说明 Node/dev 用 alias，Bun binary 用 virtualModules（两种运行形态的差异）。
- `packages/coding-agent/src/core/extensions/loader.ts:258`：`createJiti(...)` + `jiti.import(extensionPath)`，证明扩展是运行时从路径动态导入的。
- `packages/coding-agent/src/config.ts:13`：`isBunBinary` 的检测逻辑（通过 `import.meta.url` 的 `$bunfs/~BUN` 等特征），证明“Bun 编译二进制”是一个一等运行场景。
- `packages/coding-agent/test/extensions-discovery.test.ts:300`：测试覆盖“扩展可以从自身的 node_modules 解析依赖”（说明扩展并非只能用宿主内嵌依赖）。

## 相关链接
- [会话：2026-02-11](../sessions/2026-02-11.md)
- [extensions（扩展）机制：设计与实现](extensions-design-and-implementation.md)

