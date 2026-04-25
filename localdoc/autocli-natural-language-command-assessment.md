# AutoCLI 自然语言操作与参数生成能力评估

## 结论

评估时间：2026-04-25。

评估对象：

- 本机已安装命令：`autocli 0.3.8`
- 源码仓库：`https://github.com/nashsu/AutoCLI`
- 本地源码路径：`/Users/liushen/Documents/githubDevs/AutoCLI`
- 当前项目报告路径：`/Users/liushen/Documents/githubDevs/opencli/localdoc/autocli-natural-language-command-assessment.md`

结论先行：

- AutoCLI 比当前 OpenCLI 更接近“自动生成命令”的方向，因为它已经有 `explore`、`cascade`、`generate`、`generate --ai`。
- 但 AutoCLI 仍不支持“直接传入一段完整自然语言指令，然后按 Skill 安全策略完成操作”的通用 Agent 能力。
- AutoCLI 的 `generate` 支持“URL + 简短 goal -> 探索页面 API -> 生成 YAML adapter -> 保存为新命令”，这是参数/命令沉淀能力的雏形。
- AutoCLI 的 `generate --ai` 接入的是 AutoCLI.ai 云端生成接口，不是命令行参数指定任意 OpenAI-compatible / Anthropic-compatible 模型接口。
- AutoCLI 只能通过环境变量 `AUTOCLI_API_BASE` 改 AutoCLI 服务地址；这要求目标服务实现 AutoCLI.ai 的接口协议，不能直接填大模型 API base URL。
- 对“技术中台 CI 主机监控”场景，AutoCLI 能只读探索到页面和部分接口，但当前自动生成结果不够可靠：生成站点名异常、接口 URL 丢端口和路径前缀，不能直接视为可复跑命令。

一句话判断：AutoCLI 已经具备“网页 -> adapter -> 新命令”的产品化雏形，但还不是“自然语言巡检 Agent”；对内网业务系统可以作为探索和生成草稿的工具，不能直接承担稳定巡检闭环。

## 本机用法确认

`autocli -h` 显示它的定位是：

```text
AI-driven CLI tool — turns websites into command-line interfaces
```

关键内置命令包括：

- `explore`：探索网站 API surface 和 endpoint。
- `cascade`：探测 API endpoint 需要的认证策略。
- `generate`：一次性执行 explore + synthesize + 选择最佳 adapter。
- `search`：在 autocli.ai 搜索已有 adapter。
- `auth`：保存 AutoCLI token。

`autocli generate -h` 显示：

```text
Usage: autocli generate [OPTIONS] <url>

Options:
      --goal <goal>      What you want (e.g. hot, search, trending)
      --site <site>      Override site name
      --ai               Use AI (LLM) to analyze and generate adapter (requires ~/.autocli/config.json)
```

`autocli explore -h` 显示：

```text
Usage: autocli explore [OPTIONS] <url>

Options:
      --site <site>
      --goal <goal>
      --wait <wait>
      --auto
      --click <click>
```

`autocli doctor` 当前结果：

- Chrome/Chromium：通过。
- Daemon running：通过。
- Chrome extension connected：通过。
- External CLI 中 `gh`、`docker`、`kubectl` 未发现或未接入，这不影响本次网页探索能力判断。

本机没有 `~/.autocli/config.json`，因此没有 AutoCLI token；`generate --ai` 没有实测调用云端，只做源码评估。

## 源码架构

AutoCLI 是 Rust workspace，核心 crate：

- `crates/autocli-cli`：CLI 入口，使用 `clap` 动态注册 adapter 和内置命令。
- `crates/autocli-ai`：探索、认证策略探测、规则生成、AI 生成客户端。
- `crates/autocli-browser`：本地 daemon、Chrome extension bridge、AI extension proxy。
- `crates/autocli-discovery`：内置和用户 YAML adapter 发现。
- `crates/autocli-pipeline`：YAML pipeline 执行器。
- `crates/autocli-output`：table/json/yaml/csv/md 输出。

它的能力边界很清楚：核心资产仍是 YAML adapter 和确定性 pipeline；AI 主要用于生成 adapter，不是运行时持续规划每一步 UI 操作。

## 与原需求逐项对照

### 1. 能否传入一段自然语言指令进行操作

结论：不完整支持。

AutoCLI 没有类似下面这样的入口：

```bash
autocli run "根据 techmp-site-inspect 查看技术中台 CI 环境主机监控的各项信息"
autocli agent "..."
autocli nl run "..."
```

它接受的是 URL 和简短 `--goal`：

```bash
autocli explore <url> --goal <goal>
autocli generate <url> --goal <goal>
```

`--goal` 有简单的中英文归一化，源码在 `crates/autocli-ai/src/generate.rs:19-30`，支持 `search`、`hot`、`trending`、`feed`、`me`、`detail`、`comments`、`history`、`favorite` 等少数能力名。这不是完整自然语言任务解析，也不能理解 Skill 里的环境、安全、深度、输出策略。

对用户示例来说，`techmp-site-inspect` 的结构化参数应类似：

```text
target_env=ci
inspect_scope=single
target_identifier=主机监控
traversal_depth=deep
allow_write_confirm=false
```

AutoCLI 当前不会从完整自然语言中解析这些字段，也不会加载这个 Skill 的流程和风控策略。

### 2. 操作成功后能否生成新的命令行参数

结论：部分支持，但不是从“成功操作”反推出参数，而是从“页面探索结果”合成 adapter。

规则生成路径：

```text
autocli generate <url> --goal <goal>
  -> explore 页面和 API
  -> synthesize 候选 adapter
  -> 保存到 ~/.autocli/adapters/<site>/<name>.yaml
  -> 提示 autocli <site> <name>
```

源码证据：

- `crates/autocli-cli/src/main.rs:117-123` 注册了 `generate` 命令和 `--goal`、`--site`、`--ai`。
- `crates/autocli-cli/src/main.rs:1015-1022` 非 AI 路径调用 `autocli_ai::generate(...)`，成功后 `save_adapter(...)`。
- `crates/autocli-ai/src/generate.rs:135-159` 执行 `explore -> synthesize -> select`。
- `crates/autocli-ai/src/synthesize.rs:58-110` 根据探索结果生成候选 adapter。
- `crates/autocli-ai/src/synthesize.rs:331-430` 生成 YAML pipeline，包括 `navigate`、`evaluate/fetch`、`map`、`limit`。

这比 OpenCLI 当前“只有 browser init scaffold”更进一步：AutoCLI 会尝试生成完整 YAML，而不只是空骨架。

但风险也很明显：

- 它没有自动执行新 adapter 做端到端验证。
- 它不是从用户自然语言操作记录生成 replay command。
- 它不能保证生成的 endpoint、路径、字段选择正确。
- 对复杂内网站点，生成结果可能只是草稿。

### 3. AI 大模型接口地址能否通过命令行参数指定

结论：不支持以命令行参数指定任意大模型接口。

AutoCLI 的 `generate --ai` 没有 `--ai-base-url`、`--model`、`--api-key-env` 这类参数。它只要求 `~/.autocli/config.json` 里有 AutoCLI token。

源码证据：

- `crates/autocli-ai/src/config.rs:18-24` 通过 `AUTOCLI_API_BASE` 读取 AutoCLI 服务 base URL，默认 `https://www.autocli.ai`。
- `crates/autocli-ai/src/llm.rs:1-3` 写明 LLM 请求通过 AutoCLI server API，prompt 在服务端管理。
- `crates/autocli-ai/src/llm.rs:19` 固定调用 `{AUTOCLI_API_BASE}/api/ai/generate-adapter`。
- `crates/autocli-ai/src/llm.rs:40-48` 请求头使用 `Authorization: Bearer <token>`，不是读取 OpenAI/Anthropic API key。
- `crates/autocli-browser/src/daemon.rs:137-171` 和 `crates/autocli-browser/src/daemon.rs:590-596` 扩展侧 AI 生成也代理到 `{AUTOCLI_API_BASE}/api/ai/extension-generate`。

`crates/autocli-ai/src/config.rs:53-66` 定义了 `LlmConfig { endpoint, apikey, modelname }`，但搜索源码后没有发现它被 `generate --ai` 实际使用。这更像预留字段或历史遗留，并不能作为当前能力。

因此：

- 可以配置 `AUTOCLI_API_BASE=https://your-autocli-compatible-server`。
- 不能直接配置 `--ai-base-url https://api.openai.com/v1`。
- 若要私有化，需要实现 AutoCLI.ai 的服务端 API，而不是只提供一个 OpenAI-compatible endpoint。

## 技术中台 CI 主机监控实测

### explore 实测

执行受控只读探索：

```bash
autocli explore "$CI_ENV_TECHMP_URL#/devops/serverMonitor/chart" \
  --site techmp \
  --goal 主机监控 \
  --wait 3
```

为了避免把实时主机明细落盘，我只读取摘要，不保存完整 JSON。

实测摘要：

- URL：能进入 `#/devops/serverMonitor/chart?lang=zh-CN`
- title：`主机监控`
- framework：`Vue2`
- endpoint_count：`3`
- top endpoint 类型：cookie-auth JSON
- 识别到的接口类别包括主机/服务器列表、版本信息、主机指标 quota list。

这说明 AutoCLI 确实能复用浏览器登录态，访问内网技术中台页面，并发现部分后端接口。

但它没有完整覆盖页面所有信息：

- `serverMonitor.vue` 中大量趋势图依赖 `/devops/monitor/host/quota/trend`，本次普通 explore 摘要没有捕获到足够完整的趋势接口调用。
- `--auto` 和 `--click` 可能触发更多接口，但也可能误点 Tab、主机管理、进入 shell、删除/编辑等敏感入口。对技术中台这种运维系统，不应盲目启用。
- explore 输出的是 API manifest，不是巡检报告；它不会自动总结“各项信息”。

### generate 实测

为了避免污染用户真实 `~/.autocli`，我用临时 HOME 执行了非 AI 生成：

```bash
HOME=<tmp> autocli generate "$CI_ENV_TECHMP_URL#/devops/serverMonitor/chart" \
  --site techmp \
  --goal 主机监控
```

结果确实生成了 YAML adapter，并提示：

```text
autocli 211 主机监控
```

生成的 YAML 关键结构：

```yaml
site: 211
name: 主机监控
domain: 192.168.211.115
strategy: cookie
browser: true

pipeline:
  - navigate: "http://192.168.211.115:90/apollo-web/#/devops/serverMonitor/chart?lang=zh-CN"
  - evaluate: |
      (async () => {
        const res = await fetch("http://192.168.211.115/devops/monitor/config/server/list", {
          credentials: 'include'
        });
        const data = await res.json();
        return (data?.result?.data || []);
      })()
```

这次生成结果有几个关键问题：

1. `--site techmp` 没有生效，生成站点名是 `211`。源码里 `crates/autocli-cli/src/main.rs:885` 读取为 `_site`，但后续非 AI 路径 `crates/autocli-cli/src/main.rs:1017` 没有传入。
2. 生成的 fetch URL 丢了端口 `:90`，也丢了 `/apollo-web` 前缀。根因在 `crates/autocli-ai/src/synthesize.rs:264-269`，`build_templated_url()` 只拼了 `scheme + host + path`，没有包含 port；对有上下文路径或反向代理路径的内网站点容易出错。
3. 生成目标接口偏向 `/devops/monitor/config/server/list`，更像“服务器配置列表”，不是完整“主机监控各项信息”巡检。
4. 生成后没有自动验证新命令是否能运行、字段是否正确、是否覆盖趋势图。

因此，AutoCLI 在这个场景下可以生成 adapter 草稿，但不能直接说已经能完成“根据 Skill 查看 CI 主机监控各项信息，并生成可靠的新命令”。

## AI 生成路径的隐私与适配问题

`generate --ai` 的采集逻辑在 `crates/autocli-ai/src/ai_generate.rs:276-396`：

- 打开页面。
- 自动滚动。
- 从 Performance entries 中筛选 API URL。
- 最多重新 fetch 10 个 JSON API。
- 采集页面 meta、framework、globals。
- 截取主内容 HTML，最多 30000 字符。
- 把采集数据发到服务端。

这对公开站点生成 adapter 很有价值，但对技术中台这类内网运维系统有明显风险：

- 采集数据可能包含主机 IP、资源容量、系统版本、接口返回体、页面 DOM。
- 默认 AI 服务是 `https://www.autocli.ai`。
- 本地没有配置 AutoCLI token，所以本次没有实测上传。
- 即使设置 `AUTOCLI_API_BASE`，也必须确保它是可信的 AutoCLI-compatible 私有服务。

如果要用于技术中台，至少需要：

- 私有化 AutoCLI AI 服务。
- 确认采集数据脱敏。
- 限制 `capture_page_data` 不上传实时主机明细。
- 明确禁止对 stable/prdemo 这类环境执行 AI 上传。

## 与 OpenCLI 的关系

AutoCLI README 明确说它是基于 OpenCLI 的 Rust 重写，目标是单二进制、低内存、更快启动。两者理念相近，但当前能力有差异：

| 维度 | OpenCLI 当前 | AutoCLI 当前 |
| --- | --- | --- |
| 语言/分发 | TypeScript/Node | Rust 单 binary |
| 自然语言直接执行 | 不支持 | 不支持 |
| 浏览器原语 | `opencli browser` 明确暴露 | 没有等价低层 browser 子命令，更多用于 adapter/explore/generate |
| adapter 格式 | 当前主线偏 JS adapter，旧 YAML 用户 adapter 会警告 | YAML adapter 是核心格式 |
| 规则生成 adapter | `browser init` 生成骨架，`verify` 校验 | `generate` 可直接合成 YAML 草稿 |
| AI 生成 | OpenCLI 本体无通用 LLM planner | `generate --ai` 经 AutoCLI.ai 服务端生成 |
| 指定模型接口 | 不支持 | 不支持直接指定，只能 `AUTOCLI_API_BASE` 指向 AutoCLI 服务 |
| 技术中台场景 | 可用 browser 原语读取页面，但无业务 adapter | 可 explore，能生成草稿，但生成结果不可靠 |

判断上要避免一个误区：AutoCLI 的 “AI-driven” 更多指“AI 帮你生成 adapter”，不是“CLI 运行时像 Agent 一样理解并执行任意自然语言任务”。

## 是否能满足原始目标

原始目标可以拆成四项：

1. 输入自然语言指令。
2. 根据指令操作页面。
3. 成功后生成新的命令行参数或命令。
4. 大模型接口地址可通过命令行参数指定。

AutoCLI 当前满足情况：

- 第 1 项：不满足。只有 `--goal`，没有完整自然语言任务入口。
- 第 2 项：部分满足。`explore/generate` 能打开页面、滚动、捕获接口；不执行 Skill 级巡检策略。
- 第 3 项：部分满足。能生成 YAML adapter 和命令提示，但生成质量需要人工审查和修复。
- 第 4 项：不满足。没有命令行参数指定模型接口；`AUTOCLI_API_BASE` 是 AutoCLI 服务地址，不是模型地址。

对“查看技术中台 CI 主机监控各项信息”：

- 能探索页面并发现部分接口。
- 不能自动按 `techmp-site-inspect` 的流程完成深度巡检。
- 非 AI 生成的 adapter 草稿存在实际可用性问题。
- AI 生成可能更好，但默认要把页面/API 采集数据发到 AutoCLI.ai，不适合直接用于内网运维系统。

## 建议路线

### 如果想快速利用 AutoCLI

建议把 AutoCLI 当成“adapter 草稿生成器”，不是最终执行器：

1. 用 `autocli explore` 发现接口，输出只保留脱敏摘要。
2. 用 `autocli generate` 生成 YAML 草稿到临时 HOME。
3. 人工修正 `site`、endpoint port、上下文路径、字段、columns 和安全策略。
4. 手工执行新命令验证结果。
5. 验证通过后再把 adapter 放入受控目录。

对技术中台建议先做确定性命令：

```bash
autocli techmp server-monitor \
  --env ci \
  --target 主机监控 \
  --depth shallow \
  --format json
```

先不要上来做“自然语言万能巡检”。

### 如果想让 AutoCLI 真正满足原需求

需要补这些能力：

1. 新增 `autocli nl run "<instruction>"` 或 `autocli agent run "<instruction>"`。
2. 支持加载 Skill 或策略文件，把 `target_env/scope/target/depth/write-policy` 解析成结构化计划。
3. 新增本地 LLM provider 配置：
   - `--ai-provider`
   - `--ai-base-url`
   - `--ai-model`
   - `--ai-api-key-env`
   - `--dry-run`
   - `--execute`
4. 修复 `generate --site` 未传入非 AI synthesize 的问题。
5. 修复 `build_templated_url()` 丢 port 的问题，并处理内网站点上下文路径。
6. 生成 adapter 后自动 dry-run 验证。
7. 增加敏感数据脱敏与上传开关。
8. 将 `techmp-site-inspect` 这类运维 Skill 的只读/写操作策略内置到执行层，而不是只靠 prompt。

### 对当前项目的借鉴

AutoCLI 的 `explore -> synthesize -> save adapter` 这条链路值得 OpenCLI 借鉴，但不建议照搬云端 AI 路线到内网场景。

更稳妥的设计是：

```text
自然语言
  -> 本地/私有 LLM 生成可审计计划
  -> 确定性 browser/explore 操作
  -> 规则合成 adapter 草稿
  -> 自动 dry-run 验证
  -> 输出 replay command
  -> 人工确认后保存 adapter
```

其中 LLM 只做规划和草稿生成，不能绕过执行层安全策略。

## 验证边界

已执行：

- `autocli -h`
- `autocli --version`
- `autocli generate -h`
- `autocli explore -h`
- `autocli cascade -h`
- `autocli doctor`
- 克隆源码到 `/Users/liushen/Documents/githubDevs/AutoCLI`
- 源码阅读 `crates/autocli-cli`、`crates/autocli-ai`、`crates/autocli-browser`
- 对技术中台 CI 主机监控执行只读 `autocli explore` 摘要验证
- 用临时 HOME 执行非 AI `autocli generate`，未污染真实 `~/.autocli`

未执行：

- `generate --ai`：本机没有 `~/.autocli/config.json` 和 AutoCLI token；且默认会向 AutoCLI.ai 上传页面采集数据，不适合直接用于技术中台内网数据。
- Rust 单元测试：本机没有 `cargo`，执行 `cargo test -p autocli-ai` 失败，错误为 `zsh:1: command not found: cargo`。

## 最终判断

AutoCLI 可以作为“自动探索网页并生成 CLI adapter 草稿”的工具，能力明显覆盖了 OpenCLI 当前缺失的一部分参数/命令生成链路。

但对于用户提出的完整目标，它还差三层：

- 自然语言任务解析层。
- Skill 级安全策略执行层。
- 可配置大模型接口和私有化数据治理层。

因此，AutoCLI 当前不能直接完成“传入自然语言 -> 按 `techmp-site-inspect` 深度巡检技术中台 CI 主机监控 -> 成功后生成可靠命令参数 -> 通过命令行指定任意 AI 大模型接口”这一完整闭环。它最多能帮助生成初版 adapter，且在技术中台实测中生成结果还需要人工修正。
