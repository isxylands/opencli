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

## 9. 一句话概括

`opencli bilibili hot --limit 5` 的实现不是“写了个 B 站热门脚本”这么简单，而是把一个 B 站热门 API 调用封装成 YAML adapter，再接入 opencli 的统一发现、注册、浏览器会话、pipeline 执行和输出框架中。
