# OpenCLI Go 重写可行性评估

## 结论

不建议把当前 OpenCLI 做成一次性 Go 全量重写。

Go 对 CLI 的优势主要体现在单文件分发、较低冷启动成本、部署时不需要用户安装 Node 依赖。但本项目的主要复杂度不在 CLI 框架本身，而在：

- JavaScript-first adapter 生态
- Browser Bridge extension 与 daemon 通信协议
- Chrome/CDP/Electron 自动化
- 插件、用户 adapter、动态导入、输出格式、诊断和 autofix 工作流

因此，Go 全量重写的收益小于迁移成本和兼容性风险。

## 当前项目事实

- 当前仓库不是 Python 项目，而是 TypeScript/Node CLI。
- `package.json` 要求 Node.js `>=21.0.0`，入口为 `dist/src/main.js`。
- 核心 `src/**/*.ts` 约 30222 行，其中非测试约 18347 行，测试约 11875 行。
- 内置 `clis/` adapter 非测试 JS 约 64720 行，测试约 20203 行。
- `clis/` 约 712 个非测试 adapter 文件、181 个 adapter 测试文件。
- `cli-manifest.json` 当前包含 628 条命令记录。
- Chrome extension 源码约 2479 行，必须继续以浏览器扩展技术栈存在，Go 无法替代这部分。
- 运行时依赖只有 8 个，但 lockfile 里开发与文档相关依赖较多；分发体积问题更多来自 Node/npm 交付形态，而不是业务代码本身过大。

## Go 重写的真实成本

### 只重写 daemon

可行，成本中等，预计 1 到 2 周。

需要保持 HTTP/WebSocket 协议、超时、日志、状态、CSRF 防护、extension 兼容性。收益是 daemon 更容易单文件化、常驻内存可能更低，但 daemon 本身不是最主要复杂度。

### Go CLI 壳 + 继续执行 JS adapter

技术可行，但价值有限，预计 3 到 6 周。

如果 Go 只是负责命令解析，再调用 Node 执行 adapter，则用户仍然需要 Node 或内置 Node 运行时，无法兑现“无依赖”。如果嵌入 JS runtime，则 ESM、动态 import、`@jackwener/opencli/*` package exports、`fetch`、adapter API、错误模型、测试兼容都会变成新的大工程。

### Go 核心 + 保留 JS adapter 兼容层

可行性低到中，预计 2 到 4 个月。

难点不在 Cobra 或 chromedp 这类库，而在兼容现有 adapter 作者的 JS API、插件机制、用户目录热加载、manifest、diagnostic/autofix 语义。只要 JS adapter 仍是核心资产，Go 核心就必须维护一层复杂兼容 runtime。

### 全量 Go 重写 adapter

不现实，预计至少 6 到 12 人月，且风险高。

原因是 adapter 覆盖 90+ 站点，很多依赖页面结构、登录态、反爬行为、浏览器上下文 fetch、网络拦截和站点特定字段解析。重写本身只是第一步，逐站验证和后续维护才是主要成本。

## 是否值得

当前不值得。

更合理的方向是先量化用户痛点：

- 如果痛点是安装复杂：优先做 binary/package 分发方案，而不是改语言。
- 如果痛点是启动慢：继续优化 manifest fast path、lazy import、completion fast path。
- 如果痛点是 daemon 稳定性：可以考虑单独用 Go 重写 daemon 或增强现有 Node daemon。
- 如果痛点是内存：先测 CLI 冷启动、daemon 常驻 RSS、adapter 执行耗时，确认瓶颈是否真的来自 Node。
- 如果痛点是 adapter 维护：Go 不能解决站点适配复杂度，反而会提高 adapter 作者门槛。

## 建议路线

1. 保持 TypeScript/Node 作为主实现。
2. 做一次分发体验专项：评估 npm、Bun、Node SEA、pkg/nexe、安装脚本和预编译包的取舍。
3. 如果需要 Go，优先做独立 spike：只实现 daemon 协议或一个很薄的 launcher，不碰 adapter 生态。
4. 任何 Go 化决策都应先有基准数据：安装大小、冷启动时间、daemon RSS、典型 adapter 总耗时、失败率。
