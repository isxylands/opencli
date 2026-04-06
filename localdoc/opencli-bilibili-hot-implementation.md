# `opencli bilibili hot --limit 5` 实现分析

## 结论先行

这条命令本质上是一个 **YAML adapter 驱动的浏览器命令**。它不是在 Node 侧直接请求 B 站接口，而是：

1. 启动时把 `clis/bilibili/hot.yaml` 注册成 CLI 命令。
2. 通过 Commander 暴露成 `opencli bilibili hot --limit <n>`。
3. 执行时进入统一的浏览器执行框架，建立 Browser Bridge 会话。
4. 在浏览器页面上下文里执行 `fetch('https://api.bilibili.com/x/web-interface/popular?...')`。
5. 再经过 `map` 和 `limit` 两个 pipeline step，整理成统一表格输出。

它看起来像“抓页面热门视频”，但实际上更接近“借浏览器上下文去调用站内 API，然后走统一输出链路”。

---

## 1. 命令入口在哪里

直接定义在：

- `clis/bilibili/hot.yaml`

核心定义如下：

```yaml
site: bilibili
name: hot
description: B站热门视频
domain: www.bilibili.com

args:
  limit:
    type: int
    default: 20

pipeline:
  - navigate: https://www.bilibili.com
  - evaluate: |
      (async () => {
        const res = await fetch('https://api.bilibili.com/x/web-interface/popular?ps=${{ args.limit }}&pn=1', {
          credentials: 'include'
        });
        const data = await res.json();
        return (data?.data?.list || []).map((item) => ({
          title: item.title,
          author: item.owner?.name,
          play: item.stat?.view,
          danmaku: item.stat?.danmaku,
        }));
      })()
  - map:
      rank: ${{ index + 1 }}
      title: ${{ item.title }}
      author: ${{ item.author }}
      play: ${{ item.play }}
      danmaku: ${{ item.danmaku }}
  - limit: ${{ args.limit }}

columns: [rank, title, author, play, danmaku]
```

这里已经把命令的业务逻辑写全了：

- 参数：只有一个 `limit`
- 数据源：B 站热门接口 `x/web-interface/popular`
- 转换逻辑：抽出标题、作者、播放量、弹幕数
- 输出列：固定为 `rank/title/author/play/danmaku`

---

## 2. 启动时如何把这条命令注册出来

### 2.1 启动入口

- `src/main.ts`

启动后会先执行：

- `discoverClis(BUILTIN_CLIS, USER_CLIS)`
- 然后 `runCli(BUILTIN_CLIS, USER_CLIS)`

也就是说，**先发现命令，再把命令挂到 Commander 上**。

### 2.2 发现命令：开发态和发布态两条路径

- `src/discovery.ts`

`discoverClis()` 有两种模式：

1. **发布态快路径**
   - 优先读取 `cli-manifest.json`
   - 由 `src/build-manifest.ts` 在构建阶段把 YAML/TS adapter 预编译进去
   - 优点是启动时不必再解析 YAML，冷启动更快

2. **开发态回退路径**
   - 直接扫描 `clis/` 目录
   - 对 `clis/bilibili/hot.yaml` 调用 `registerYamlCli()`
   - 用 `js-yaml` 读取 YAML，再转成内部 `CliCommand`

所以从“源码仓库如何实现”这个角度看，开发时更接近：

`src/main.ts` -> `discoverClis()` -> `registerYamlCli("clis/bilibili/hot.yaml")`

而从“安装后的最终运行形态”看，往往会变成：

`src/main.ts` -> `discoverClis()` -> `loadFromManifest()`

### 2.3 YAML 如何转成内部命令对象

- `src/discovery.ts`
- `src/yaml-schema.ts`
- `src/registry.ts`

`registerYamlCli()` 会把 YAML 转成内部 `CliCommand`：

- `site = bilibili`
- `name = hot`
- `domain = www.bilibili.com`
- `args = [{ name: 'limit', type: 'int', default: 20, ... }]`
- `pipeline = [...]`
- `columns = [...]`

一个关键细节是：

- YAML 没显式写 `strategy`
- 框架会按默认规则推导为 `cookie`

来源是 `src/discovery.ts`：

- 如果 `browser !== false`，默认策略不是 `public`，而是 `cookie`

这会直接影响后面的执行方式。

---

## 3. CLI 层怎么把它变成 `opencli bilibili hot --limit 5`

### 3.1 注册 Commander 子命令

- `src/cli.ts`
- `src/commanderAdapter.ts`

`runCli()` 最终会调用：

- `createProgram(...).parse()`

在 `createProgram()` 里，动态 adapter 会走：

- `registerAllCommands(program, siteGroups)`

接着：

- 为 `bilibili` 创建一级命令组
- 为 `hot` 创建二级子命令

于是才有了这条命令路径：

`opencli bilibili hot`

### 3.2 `--limit 5` 是怎么进来的

`registerCommandToProgram()` 会把 YAML 里的参数注册成 Commander option。

对 `limit` 来说，最终效果相当于：

- `--limit [value]`
- 默认值 `20`

用户输入：

```bash
opencli bilibili hot --limit 5
```

在 Commander 层先拿到的通常还是字符串值 `"5"`，之后再交给统一执行器做类型收敛。

---

## 4. 执行阶段的真实调用链

完整链路可以概括为：

```text
src/main.ts
  -> discoverClis()
  -> runCli()
  -> createProgram().parse()
  -> registerCommandToProgram(...).action(...)
  -> executeCommand(cmd, kwargs, verbose)
  -> browserSession(...)
  -> runCommand(...)
  -> executePipeline(page, pipeline, { args })
  -> output.render(...)
```

### 4.1 参数校验和类型转换

- `src/execution.ts`

`executeCommand()` 第一件事就是调用：

- `coerceAndValidateArgs()`

这里会把：

- `limit: "5"` 转成 `limit: 5`

这一步很关键，因为 YAML pipeline 里后面会把 `args.limit` 用到：

- URL 模板替换
- 最终结果截断

### 4.2 为什么一定会走浏览器会话

- `src/capabilityRouting.ts`
- `src/execution.ts`

`shouldUseBrowserSession(cmd)` 对这个命令会返回 `true`，原因有两层：

1. 命令默认是浏览器命令
2. pipeline 中包含 `navigate` / `evaluate`，这些都属于浏览器 step

因此执行前会：

- 检查 Browser Bridge 状态
- 创建浏览器会话
- 在 `site:bilibili` workspace 里拿到一个 `page`

### 4.3 统一框架先做一次“预导航”

- `src/execution.ts`

这个命令的 `strategy` 是 `cookie`，并且定义了：

- `domain: www.bilibili.com`

所以 `resolvePreNav(cmd)` 会先得到：

- `https://www.bilibili.com`

然后在真正跑 pipeline 之前，框架先执行一次：

- `page.goto('https://www.bilibili.com')`

这么做的目的是：

- 让浏览器先进入目标域名
- 后续 `fetch(..., { credentials: 'include' })` 能自动带上该站 cookie

这不是 `hot.yaml` 自己写的逻辑，而是整个执行框架的统一行为。

---

## 5. pipeline 每一步具体做了什么

### Step 1: `navigate`

- 实现：`src/pipeline/steps/browser.ts` 中的 `stepNavigate`

执行：

```yaml
- navigate: https://www.bilibili.com
```

效果就是再次打开 B 站首页。

注意：这一步和上面的“预导航”目标相同，因此存在一定程度的重复。

### Step 2: `evaluate`

- 实现：`src/pipeline/steps/browser.ts` 中的 `stepEvaluate`
- 模板渲染：`src/pipeline/template.ts`

这一层最关键。

`stepEvaluate` 会先用模板引擎把：

- `https://api.bilibili.com/x/web-interface/popular?ps=${{ args.limit }}&pn=1`

渲染成真实字符串，例如：

- `https://api.bilibili.com/x/web-interface/popular?ps=5&pn=1`

然后把整段 JS 传给：

- `page.evaluate(js)`

真正执行位置不是 Node 进程，而是 **浏览器页面上下文**。

执行结果是一个数组，每项大概长这样：

```json
{
  "title": "...",
  "author": "...",
  "play": 123456,
  "danmaku": 789
}
```

### Step 3: `map`

- 实现：`src/pipeline/steps/transform.ts` 中的 `stepMap`

`map` 会遍历上一步返回的数组，并对每一项应用模板：

- `rank: ${{ index + 1 }}`
- `title: ${{ item.title }}`
- `author: ${{ item.author }}`
- `play: ${{ item.play }}`
- `danmaku: ${{ item.danmaku }}`

这里的关键上下文变量来自 `src/pipeline/template.ts`：

- `args`：命令参数
- `item`：当前元素
- `index`：当前下标
- `data`：整批上游数据

所以 `rank` 不是接口直接返回的，而是本地二次生成的序号。

### Step 4: `limit`

- 实现：`src/pipeline/steps/transform.ts` 中的 `stepLimit`

最后再执行：

```yaml
- limit: ${{ args.limit }}
```

也就是：

- 再对映射后的数组做一次 `slice(0, limit)`

对于 `--limit 5`，结果最终被截成前 5 项。

---

## 6. 最终输出是怎么变成表格的

- `src/commanderAdapter.ts`
- `src/output.ts`

命令执行结果返回后，`registerCommandToProgram()` 会调用：

- `renderOutput(result, { columns, title, elapsed, source, ... })`

这里会把 `columns` 固定为 YAML 里声明的顺序：

- `rank`
- `title`
- `author`
- `play`
- `danmaku`

默认输出格式是 `table`，所以终端里通常看到的是表格。

补充一个经常被忽略的行为：

- 如果不是 TTY，且用户没显式传 `-f`
- `src/output.ts` 会自动把默认格式从 `table` 降成 `yaml`

也就是说，这条命令在脚本/管道环境里的默认输出未必是表格。

---

## 7. 这个实现真正值得关注的关键点

### 7.1 它依赖浏览器，但核心数据其实来自 API

这是最重要的一点。

`bilibili hot` 看起来是“浏览器抓取”，但真正拿数据的核心并不是 DOM，而是：

- `https://api.bilibili.com/x/web-interface/popular`

浏览器在这里主要承担两个职责：

1. 提供统一的站内执行环境
2. 通过 `credentials: 'include'` 自动带 cookie

换句话说，这个命令的核心思想是：

- **用浏览器保证站点上下文**
- **用接口拿结构化数据**

而不是直接解析页面 HTML。

### 7.2 `limit` 被用了两次

`limit` 同时影响：

1. API 请求参数 `ps=${{ args.limit }}`
2. pipeline 末尾的 `limit` step

这说明实现采取了“前端请求尽量少拿，后端结果再兜底截断”的策略。

从纯功能角度看，末尾这次 `limit` 有点冗余；但从鲁棒性角度看，它能防止接口返回数量与预期不一致。

### 7.3 存在重复导航

这个命令会有两次导航到 B 站首页：

1. 执行框架的预导航
2. YAML pipeline 里的 `navigate`

这不是功能错误，但说明 adapter 层和框架层存在一部分职责重叠。

如果后续要优化启动速度，这里是很明显的切入点。

### 7.4 它只拿第一页

接口 URL 写死了：

- `pn=1`

所以这条命令本质上永远只读“热门第一页”，不支持翻页，也没有做更多分页聚合。

这意味着：

- `--limit` 调大时，本质上只是让第一页返回更多项
- 它不是一个真正意义上的全量遍历器

### 7.5 这个接口未必真的需要登录，但框架仍按浏览器命令处理

从这份 adapter 代码本身看：

- 它没有显式依赖用户专属数据
- 也没有读取登录态专属字段

但整个命令仍然要求浏览器环境、扩展桥接、站点上下文。

这说明当前框架更重视：

- 统一执行模型
- 统一 cookie/风控处理方式

而不是为每个公开接口单独做最轻量实现。

这是工程上偏保守、偏一致性的选择，不是最省成本的选择。

---

## 8. 关键源码索引

如果你要继续深挖，优先看这些文件：

- `clis/bilibili/hot.yaml`
- `src/main.ts`
- `src/discovery.ts`
- `src/build-manifest.ts`
- `src/cli.ts`
- `src/commanderAdapter.ts`
- `src/execution.ts`
- `src/capabilityRouting.ts`
- `src/pipeline/executor.ts`
- `src/pipeline/template.ts`
- `src/pipeline/steps/browser.ts`
- `src/pipeline/steps/transform.ts`
- `src/output.ts`

---

## 9. 如果 B 站改版，这条命令会不会失效

会，但不是“网页一改版就一定失效”，而是要看 **改的是哪一层**。

### 9.1 哪些改版大概率不影响它

这条命令不是靠 DOM selector 抓热门卡片，而是靠浏览器里的 `fetch()` 去请求：

- `https://api.bilibili.com/x/web-interface/popular`

所以如果只是这些变化：

- 首页视觉样式改版
- 热门列表卡片 DOM 结构变化
- CSS class、布局、前端组件重构

通常 **不会直接影响** `bilibili hot`，因为它几乎不依赖页面结构解析。

### 9.2 哪些改版很可能让它失效

下面这些变化就更危险：

1. B 站把热门接口换了地址
   - 例如 `x/web-interface/popular` 下线、迁移或参数变化

2. 接口返回结构改了
   - 例如 `data.data.list` 不存在了
   - 或 `item.owner.name`、`item.stat.view` 这些字段改名/挪位置了

3. 请求校验升级了
   - 例如需要额外签名、动态 token、wbi 参数、时间戳、nonce
   - 这类变化对 YAML adapter 很不友好

4. 浏览器上下文不再足够
   - 例如接口改成更严格的风控策略
   - 或请求前必须先跑一段页面 JS 才能拿到必要参数

5. 首页预导航本身失效
   - 例如 `https://www.bilibili.com` 被强制跳挑战页、风控页或地区页
   - 那后续 `evaluate(fetch(...))` 也可能跟着失效

### 9.3 失效后是不是要“重新注册”

严格说，**一般不叫重新注册**，而是要区分运行形态。

#### 情况 A：你在当前源码仓库里开发/运行

如果你改的是：

- `clis/bilibili/hot.yaml`

那么通常只需要：

1. 修改 YAML adapter
2. 重新运行 `opencli` 或开发命令

因为启动时会重新执行 discovery，把 adapter 重新读进 registry。

也就是说，在源码开发态下，本质是“重启后重新发现”，不是手工做一遍“注册动作”。

#### 情况 B：你运行的是构建后的发布产物

这时通常会优先走：

- `cli-manifest.json`

所以如果只改源码里的 YAML，但没有重新构建，那么运行时可能还在吃旧 manifest。

这种情况下正确动作是：

1. 修改 `clis/bilibili/hot.yaml`
2. 重新构建产物
3. 重新生成 manifest
4. 用新的构建结果运行或重新安装包

否则你会以为“adapter 改了怎么没生效”，其实是 manifest 还没更新。

#### 情况 C：你用的是用户目录或插件目录里的 adapter

如果 adapter 放在：

- `~/.opencli/clis`
- `~/.opencli/plugins/...`

那也通常是修改后重新启动命令，让 discovery 再加载一次。

所以从 opencli 机制看，真正需要“register”的通常不是这种内建/本地 adapter 更新，而更像：

- 外部 CLI 的注册
- 插件安装/启用

把 `bilibili hot` 失效问题理解成“要不要重新注册”其实有点偏题，重点应该是：

- **adapter 要不要改**
- **改完后当前运行形态会不会重新加载新定义**

### 9.4 这条命令在 B 站改版下的脆弱点排序

按这份实现看，脆弱点大致从高到低是：

1. 接口地址和参数契约
2. 返回 JSON 结构
3. 风控/签名要求
4. 首页预导航可用性
5. 页面 DOM 样式

这和很多“纯页面抓取”命令正好相反。

也就是说，`bilibili hot` 对“前端视觉改版”不敏感，但对“接口契约改版”很敏感。

### 9.5 如果真改版了，优先怎么修

优先修复顺序建议是：

1. 先确认失效点是在 `navigate` 还是 `evaluate(fetch(...))`
2. 如果只是接口地址或字段变了，先改 YAML
3. 如果开始需要复杂签名、动态 token、重试逻辑，别硬撑 YAML，直接改成 TS adapter

这里给一个跳出当前思路的建议：

如果 B 站后续把热门接口也收紧到类似带签名参数、动态计算校验值的模式，那么继续把逻辑塞在 YAML 的 `evaluate` 里会越来越难维护。这时更合理的做法不是“继续补 YAML”，而是把它升级成：

- `clis/bilibili/hot.ts`

原因是 TS adapter 更适合承载：

- 签名算法
- 分步请求
- 更细的错误分类
- 调试日志
- 回退策略

否则你会得到一个“看起来还是声明式，其实已经半脚跨进脚本地狱”的 YAML。

---

## 10. 新网站支持的三种层级

很多人看到 opencli 的宣传语“让任何网站变成 CLI”，会误以为：

- 只要给一个新网址
- opencli 就会立刻自动生成稳定可复用的正式命令

这理解过头了。

更准确的说法是，opencli 对新网站的支持分 3 个层级，而且这 3 层的适用场景完全不同。

### 10.1 第一层：立即可操作

这是门槛最低的一层。

前提通常只有两个：

1. 目标网站能在本机 Chrome/Chromium 中打开
2. 你已经有可复用的登录态，或者至少能手工登录

这时可以直接使用 opencli 的浏览器原语能力：

- `opencli operate open <url>`
- `opencli operate state`
- `opencli operate click <index>`
- `opencli operate type <index> <text>`
- `opencli operate get value <index>`

这一层的本质不是“已有适配器”，而是：

- **先把网站变成一个可被 AI/命令行逐步操控的浏览器会话**

它适合：

- 临时排查
- 一次性读取页面
- 人工辅助式自动化
- 尚未沉淀成正式命令的企业内网页面

它不适合：

- 高频复用
- 团队标准化
- 需要稳定结构化输出的场景

### 10.2 第二层：可探索、可半自动生成

当你不满足于“能点”，而是希望：

- 找出这个站点背后的 API
- 判断该用 public/cookie/header/intercept/ui 哪种策略
- 尝试生成一个初版命令

就进入第二层。

对应能力通常是：

- `opencli explore <url> --site <name>`
- `opencli generate <url> --goal "<goal>"`

这一层的本质是：

- **从浏览器交互和抓包中，提炼出可程序化的站点能力**

但对企业私域网站要保持清醒：

- 这类站点经常有 SSO
- 菜单按权限裁剪
- 依赖内网/VPN
- 请求里有动态 token、签名、上下文 header

所以第二层通常能帮你：

- 缩短探索时间
- 找到切入点
- 产出原型

但很难保证：

- 一次生成就可长期稳定使用

### 10.3 第三层：正式沉淀为 adapter / plugin

这是 opencli 真正“支持一个新网站”的工程化完成态。

也就是把能力沉淀为：

- YAML adapter
- TS adapter
- 或 plugin

一旦沉淀完成，用户就能稳定地跑：

- `opencli <site> <command>`

这一层适合：

- 高频重复任务
- 需要结构化输出
- 团队协作
- 需要版本管理、测试、发布

对于企业私域网站，这通常才是长期可维护的正确落点。

### 10.4 `techmp-site-inspect` 属于哪一层

`techmp-site-inspect` 很容易让人误判。

它**不是** opencli 的内建站点 adapter，也不是简单的 `opencli operate` 包装，而是：

- 一套更高层的 Agent 巡检工作流

它的特点是：

- 依赖 `chrome-devtools`
- 强制 1712x1080 视口
- 有环境映射（CI/Local/Stable/PRDemo/Custom）
- 有登录态识别和复用逻辑
- 有单页/模块/全站三种巡检策略
- 有分级安全与写操作风控

这说明它更接近：

- **面向企业后台巡检的流程化技能**

而不是：

- **面向站点能力沉淀的 opencli adapter**

所以如果拿“技术中台 CI 环境”举例，合理的使用顺序通常是：

1. 先用 `techmp-site-inspect` 做巡检、摸底、识别风险点
2. 再把其中稳定且高频的动作沉淀成 opencli adapter

如果直接跳到第 3 层而跳过巡检摸底，往往会犯两个错误：

1. 过早把脆弱页面流程固化成命令
2. 把本该属于“巡检工作流”的逻辑塞进 adapter，导致 adapter 既重又脆

---

## 11. YAML adapter 和 TS adapter 的区别

这是给新网站落地命令时最容易选错的一步。

### 11.1 本质区别

两者的本质差异不是“一个写 YAML，一个写 TypeScript”这么表面，而是：

- **YAML adapter 适合声明式 pipeline**
- **TS adapter 适合编程式控制逻辑**

也就是：

- YAML 更像“配置驱动的执行图”
- TS 更像“完整的程序”

### 11.2 YAML adapter 的特点

YAML adapter 一般由这些组成：

- 元信息：`site`、`name`、`description`、`domain`
- 参数定义：`args`
- 执行流程：`pipeline`
- 输出列：`columns`

典型 step 包括：

- `navigate`
- `evaluate`
- `fetch`
- `map`
- `filter`
- `sort`
- `limit`

优点：

1. 上手快
2. 结构清晰
3. 适合简单数据抽取
4. 适合 API 调用 + 简单映射
5. 更容易让非核心开发者读懂

缺点：

1. 复杂控制流很难写
2. 一旦 `evaluate` 里塞太多 JS，就会快速失控
3. 不适合复杂签名、重试、分页聚合、异常分支
4. 调试体验弱于 TS

最适合的场景：

- 公开 API
- Cookie 即可访问的简单接口
- 单页面数据读取
- 轻量 DOM 抽取

### 11.3 TS adapter 的特点

TS adapter 一般通过 `cli({...})` 注册命令，并在 `func` 里写完整逻辑。

优点：

1. 可写复杂逻辑
2. 可做多步请求和分页
3. 可做签名计算、动态 token 处理
4. 可做更细的错误分类和日志
5. 可复用 helper 函数和站点工具库
6. 维护性在复杂场景下远好于 YAML

缺点：

1. 代码量更大
2. 上手成本更高
3. 对贡献者要求更高
4. 需要更认真地做测试和边界处理

最适合的场景：

- 企业私域站点
- SSO/复杂鉴权
- 多接口聚合
- 深分页
- 拦截器
- 写操作
- 复杂 UI 自动化

### 11.4 在框架里的运行差异

从运行机制看，两者也有差别。

#### YAML adapter

- 启动时会被解析成 `CliCommand`
- 主要承载 `pipeline`
- 执行时走 `executePipeline()`

在发布态下，YAML 还可能被预编译进 manifest，减少运行时解析成本。

#### TS adapter

- 启动时通过 `cli({...})` 注册
- 执行逻辑主要在 `func`
- 发布态下通常以 lazy-load 方式在首次执行时再加载模块

所以从工程视角看：

- YAML 偏“数据定义”
- TS 偏“代码模块”

### 11.5 什么时候该从 YAML 升级到 TS

如果你出现以下任意一种情况，就该认真考虑升级：

1. `evaluate` 里已经塞了大段 JS
2. 需要循环分页并合并结果
3. 需要请求签名、token 刷新、复杂 header
4. 需要根据不同错误做不同回退
5. 需要多来源聚合
6. 需要写操作或强风控验证

一句话判断：

- **简单读取用 YAML**
- **复杂行为用 TS**

继续死撑 YAML 的典型后果是：

- 表面上还是“声明式配置”
- 实际上已经在 `evaluate` 里偷偷长成一段难以维护的脚本

那时你得到的不是简洁，而是伪简洁。

---

## 12. 面向企业私域网站的落地建议

如果目标是一个新的企业私域网站，例如技术中台 CI 环境，更合理的落地顺序通常是：

1. **先巡检，不要先沉淀**
   - 用 `techmp-site-inspect` 这类工作流先摸清登录方式、菜单结构、风险按钮、页面稳定性

2. **再探索，不要先抽象**
   - 用 `opencli operate` / `opencli explore` 看真实交互和真实请求

3. **最后再沉淀**
   - 稳定简单能力写成 YAML adapter
   - 复杂业务流程直接写 TS adapter

对技术中台这类系统，典型的拆分方式通常是：

- 适合 YAML 的：
  - 读取模块菜单
  - 拉取简单列表
  - 调用已有 JSON 接口并格式化输出

- 适合 TS 的：
  - 登录态处理
  - 复杂筛选页面
  - Tab 深度巡检
  - 写操作验证
  - 有安全策略和环境分级的流程

也就是说，`techmp-site-inspect` 这种场景，本质上更像：

- “先用工作流技能摸清现场”

而不是：

- “直接一把梭把整个系统做成单个 opencli 命令”

后者看起来省事，实际上维护成本最高。

---

## 13. 以 Gemini / ChatGPT 为例，第三方网站要如何添加支持

先纠正一个容易混淆的点：

- `gemini/*` 在当前仓库里是**网站支持**
- `chatgpt/*` 在当前仓库里**不是 chatgpt.com 网站支持**，而是 **ChatGPT macOS Desktop App** 自动化

所以如果你说“像 ChatGPT、Gemini 这样的第三方网站”，在当前代码基里真正对标的现成样本其实是：

- `Gemini Web`

而不是：

- `ChatGPT Desktop`

这不是字眼问题，而是接入路线完全不同。

### 13.1 现成样本分别是什么

#### Gemini

- 代码：`clis/gemini/ask.ts`
- 文档：`docs/adapters/browser/gemini.md`
- 类型：**浏览器网站 adapter**

特点：

- `strategy: Strategy.COOKIE`
- `browser: true`
- 依赖 Chrome 登录态
- 通过 Browser Bridge 连接浏览器
- 通过页面 DOM 和页面上下文辅助完成“发消息 / 等待响应 / 提取回复”

Gemini 说明了一类常见第三方网站的接入方式：

- 没有直接暴露给 opencli 的稳定公开 API
- 但可以复用已登录浏览器会话
- 页面本身又是高度动态的交互式应用

这类站点通常更适合：

- **TS adapter**

而不是 YAML。

#### ChatGPT

- 代码：`clis/chatgpt/ask.ts`
- 文档：`docs/adapters/desktop/chatgpt.md`
- 类型：**macOS 原生桌面自动化**

特点：

- `browser: false`
- 不依赖 Browser Bridge
- 默认依赖 AppleScript + 剪贴板 + Accessibility Tree
- 目标是桌面应用，不是网站 DOM

这意味着：

- 你不能把当前 `chatgpt/*` 直接当成“网站 adapter 范本”

如果以后真要加 `chatgpt.com` 网站支持，路线应该更接近 Gemini，而不是当前这个 ChatGPT Desktop 实现。

### 13.2 给一个第三方网站新增支持，推荐流程

以 `gemini.google.com` 这类站点为样本，更合理的流程通常是：

1. **先确认目标是网站，不是桌面应用**
   - 这是第一步，不要跳过

2. **先做一次可操作验证**
   - 确认 Chrome 能打开
   - 确认登录态可复用
   - 确认 Browser Bridge 正常

3. **再探索真实数据来源**
   - 看有没有公开 API
   - 看 Cookie 带上后能否直接 `fetch`
   - 看是否需要额外 header、token、签名
   - 看是否只能走 UI

4. **最后决定 adapter 形态**
   - 简单接口读取：YAML
   - 状态化交互、流式输出、复杂页面：TS

### 13.3 对第三方网站，真正要先回答的不是“能不能支持”，而是“走哪条策略”

对于新网站，优先要回答这几个问题：

1. 有稳定公开 API 吗
2. 没有公开 API 时，带 Cookie 的浏览器 `fetch` 能拿到数据吗
3. 需要额外 header / CSRF / Bearer 吗
4. 请求有没有复杂签名
5. 如果接口路线走不通，UI 自动化能否可靠完成任务

这对应 opencli 里常见的策略选择：

- `public`
- `cookie`
- `header`
- `intercept`
- `ui`

### 13.4 对 ChatGPT / Gemini 这类 AI 网站，默认应该偏向 TS adapter

这是一个很重要的工程判断。

这类网站通常有这些共性：

- 页面状态复杂
- 输入框不是普通表单
- 回复是流式生成
- 有“继续生成”“停止”“重试”“模型切换”等状态分支
- 可能插入登录、配额、同意协议、风控挑战 UI

所以即便理论上能在 YAML 的 `evaluate` 里硬写，也不建议。

对这类网站，更合理的默认策略是：

- **先做 TS adapter**

原因很现实：

- 你迟早要写等待逻辑
- 你迟早要处理异常分支
- 你迟早要写 helper

继续硬塞 YAML，通常只是在把复杂度藏起来，而不是消灭复杂度。

### 13.5 如果要新增 `chatgpt.com` 网站支持，推荐怎么做

如果未来要给 **ChatGPT Web** 加支持，推荐路线大致是：

1. 不要从 `clis/chatgpt/*` 复制
   - 因为那是桌面 App 自动化

2. 反而应该优先参考：
   - `clis/gemini/ask.ts`
   - `clis/gemini/utils.ts`

3. 具体实现上，大概率会是：
   - `browser: true`
   - `strategy: Strategy.COOKIE` 或 `Strategy.UI`
   - 通过页面 DOM 找 composer
   - 发送 prompt
   - 轮询或观察流式输出区域
   - 提取最终回复

4. 如果发现网络请求可复现且稳定，再考虑往 API 侧靠
   - 否则直接按 UI/页面状态机实现

所以对 `chatgpt.com` 网站来说，最像的现成参考其实不是仓库里的 `chatgpt/*`，而是：

- **Gemini 的网页 adapter 路线**

---

## 14. 网站支持和桌面应用支持，有什么异同

这个问题不能简单回答成“一个是网页，一个是 App”。

更准确地说，当前仓库里至少有 3 种不同的接入模型：

1. **网站 + Browser Bridge**
2. **Electron 桌面应用 + CDP**
3. **原生桌面应用 + AppleScript / Accessibility**

### 14.1 共同点

无论是网站还是桌面应用，只要它最终被接成 opencli 命令，都会尽量复用同一套上层框架：

- `registry`
- `commanderAdapter`
- `execution`
- `output`

也就是说，它们在“命令发现、参数解析、统一输出”这一层是共用的。

这是 opencli 架构里一个很重要的统一性来源。

### 14.2 网站 + Browser Bridge

代表样本：

- `bilibili/*`
- `gemini/*`
- `zhihu/*`

特点：

- 运行目标是 Chrome 里的网页
- 通过 Browser Bridge + daemon 与浏览器通信
- 复用现有网页登录态
- `domain`、`cookie`、预导航都很重要

这类支持最容易受影响的地方通常是：

- 网站 API
- 页面 DOM
- 登录态/风控

### 14.3 Electron 桌面应用 + CDP

代表样本：

- `codex/*`
- `cursor/*`

特点：

- 目标是 Electron App 内部的 Chromium 渲染进程
- 通过 CDP 连接，不走普通网页 Browser Bridge 路径
- 通常需要 `--remote-debugging-port`
- `src/electron-apps.ts` 负责内建应用登记
- `src/launcher.ts` 负责探测、启动、连接

但它和网站支持又有一个非常关键的相似点：

- **因为 Electron 内部仍然是 DOM/UI，所以很多交互方式依然像“自动化网页”**

这就是为什么 `codex/ask.ts` 仍然会写：

- `page.evaluate(...)`
- `page.pressKey('Enter')`
- 轮询页面内容变化

也就是说：

- **Electron 桌面应用在“传输层”上更像桌面**
- **在“交互层”上又更像网页**

这是它最容易被误解的地方。

### 14.4 原生桌面应用 + AppleScript / Accessibility

代表样本：

- 当前仓库里的 `chatgpt/*`

特点：

- 不通过 Browser Bridge
- 不通过 CDP
- 不依赖网页 DOM
- 直接使用系统自动化能力

这类支持最容易受影响的地方不是网页结构，而是：

- macOS 权限
- App 菜单和可访问性树
- 原生窗口焦点
- 剪贴板/快捷键行为

所以它和网站支持的差异其实比 Electron 还大。

### 14.5 三类模式对比

| 模式 | 代表 | 连接方式 | 主要交互对象 | 典型风险 |
|------|------|----------|--------------|----------|
| 网站 | Gemini / Bilibili | Browser Bridge | 网页 DOM / 页面上下文 / 站内 API | DOM 改版、接口变更、Cookie/风控 |
| Electron 桌面 | Codex / Cursor | CDP | App 内部 DOM / Chromium 渲染树 | 调试端口、App UI 变更、选择器失效 |
| 原生桌面 | ChatGPT Desktop | AppleScript / AX | 原生窗口 / Accessibility Tree | 系统权限、控件层级变化、焦点问题 |

### 14.6 对新增支持的真正决策顺序

所以面对一个新目标，正确顺序不是直接问：

- “它是网站还是桌面？”

而是应该问：

1. 它的运行载体是什么
2. 它暴露给自动化的控制面是什么
3. 最稳定的数据来源是 API、DOM、还是系统辅助功能
4. 哪一层的变更成本最低、维护性最好

这是新增支持时比“会不会点页面”更关键的判断。

---

## 15. 关于本仓库后续记录方式的建议

这次讨论暴露了一个很实际的问题：

- 一开始文档只是在分析 `bilibili hot`
- 但随着讨论深入，内容已经扩展到：
  - 新网站支持层级
  - YAML vs TS adapter
  - 企业私域网站落地策略
  - 第三方网站与桌面应用接入模型

这说明后续更合理的做法应该是：

- `localdoc/` 不只放“单个命令分析”
- 也要放“跨命令、跨站点、跨架构的设计结论”

否则有价值的讨论会散落在对话里，而不是沉淀成仓库知识。

---

## 16. 一句话概括

`opencli bilibili hot --limit 5` 的实现不是“写了个 B 站热门脚本”这么简单，而是把一个 B 站热门 API 调用封装成 YAML adapter，再接入 opencli 的统一发现、注册、浏览器会话、pipeline 执行和输出框架中。
