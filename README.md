# DSH 插件接口手记

写这个东西的原因很具体：插件写完了，装上了，钩子不触发；或者触发了，但什么都不做，日志里还看不出异常。

下面每一条都是踩过之后记下来的做法。按顺序看，能少走不少弯路。

内容以中文为主，每节末尾有一句英文摘要。

---

## 一、先分清「没反应」是哪一种

「装了没效果」至少有四种，修法完全不同。不先分开，会一直在错的那层改代码。

| 现象 | 说明 | 先查什么 |
|---|---|---|
| 连模块都没加载 | 打包进去了，但没被 import | bundle 补丁里有没有 `- insert: - id/name` 这一项（空数组 `[]` 是什么都不注册） |
| 加载了，但 `apply()` 没跑 | 依赖没解析，或者配置校验抛错 | 顶层探针有输出、`apply()` 里的探针没有 |
| `apply()` 跑了，钩子没注册 | 注册时抛错，被 try 吞了 | 把注册的返回值打出来 |
| 钩子注册了，没被调用 | 事件没触发，或者被上游拦掉了 | 钩子入口第一行打探针 |

*English: four distinct failure modes, each with a different first check. Separate them before touching code.*

---

## 二、探针优先

不确定一段代码有没有执行，就在里面写一行文件。别靠推理。

```js
import { appendFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
const PROBE = join(tmpdir(), 'probe.log');
function probe(m) { try { appendFileSync(PROBE, `${new Date().toISOString()} ${m}\n`); } catch {} }
```

模块级探针要写在顶层，比 `apply()` 更早：

```js
probe(`MODULE LOADED pid=${process.pid} cwd=${process.cwd()} file=${import.meta.url}`);
```

| 日志 | 结论 |
|---|---|
| 连这条都没有 | 模块没被 import，去查加载配置 |
| 有这条，没有 `apply()` | 加载了但没初始化，查返回值或依赖 |
| 有 `apply()`，钩子没触发 | 钩子没挂上，查注册 |
| 钩子触发了但行为不对 | 逻辑问题，在钩子内部加探针 |

*English: probe first, infer later. A module-level probe separates "not loaded" from "loaded but inert".*

---

## 三、访问服务必须声明 `inject`

两种写法都会抛注入门错误，`ctx.get()` 也不能绕：

```js
ctx.shell            // ✗ cannot get property "shell" without inject
ctx.get('shell')     // ✗ 同样抛
```

正确做法是声明依赖：

```js
export const inject = ['userQuestions'];

export function apply(ctx) {
  ctx.userQuestions.ask(...);
}
```

`inject` 同时是能力边界：声明什么，插件才有什么。中枢类插件只声明 `userQuestions`、不声明 `tools` 和 `shell`，就动不了手。

这个错误**只在真机暴露**，离线 mock 的 ctx 没有注入门，怎么写都不报错。所以单测里要断言它：

```js
assert.ok(Array.isArray(mod.inject));
assert.ok(mod.inject.includes('userQuestions'));
assert.ok(!mod.inject.includes('tools'));   // 不该有的别声明
```

*English: declare `inject`, access directly. `ctx.get()` does not bypass the gate, and mocks never reproduce the error.*

---

## 四、失效方向要选，而且只有两类

读不到状态时怎么办，取决于这个插件是什么东西。

| 失效方式 | 代价 |
|---|---|
| 该沉默时报警 | 多一条提醒，看得见，改得掉 |
| 该报警时沉默 | 插件装着却什么都不做，日志里看不出异常 |

记账、提醒、预算这一类，后者严重得多，所以要**倾向于生效**（读不到就按"最需要提醒"处理）。
安全边界才该 fail-closed。

这里有个坑值得单独说：一旦选了"读不到就沉默"，你会得到一个几乎无法排查的状态——插件在跑，钩子在挂，什么都正常，只是永远不动作。

*English: choose the failure direction deliberately. Accounting tools fail toward acting; only security boundaries fail closed.*

---

## 五、钩子返回什么

三个钩子的返回结构不一样，混用会静默失效。

`tools/pre-execute` —— 决定这次调用放不放：

```js
return next();                                   // 放行
return { kind: 'ask', reason: '...' };           // 交给审批通道
return { kind: 'block', feedback: '...' };       // 否决，feedback 回到模型
```

`tools/post-execute` —— 不要替换 `content`，只追加：

```js
return { ...downstream, additionalContexts: [notice, ...(downstream.additionalContexts ?? [])] };
```

`downstream` 本身是 `{kind:'block'}` 时，要原样保留它，再把 `additionalContexts` 带上，否则会吞掉上游的否决。

`agent/pre-step` —— 判断"这是不是新的一轮用户消息"，要区分来源：

```js
if (m?.source?.kind !== 'user') continue;   // 自己注入的 notice 用别的 kind
```

**幂等性**：钩子可能被重放。计数类状态建议用 `WeakMap` 挂在 agent 上，agent 释放即回收，不用手动清理。

`turn/end.reason` 目前是六态：`completed / aborted / blocked / error / max-tokens / interrupted`，类型注释写明是 **merge-extensible**，插件可以合并新变体。下游不消费时它仍然会传播，所以先加变体、再等核心支持是可行的。

*English: the three hooks return different shapes. post-execute must preserve the downstream verdict. turn/end reasons are merge-extensible.*

---

## 六、终态比超时重要

这一节是最容易出死进程的地方。

只设超时、不设终态，会得到一个很糟的状态：**会话停在那里，从外面看和"正在干活"完全一样**。没有报错，没有终态事件，只有不再前进。

要做的是给每一条等待路径定义一个终态，并且区分有没有人在场：

| 场景 | 动作 | 依据 | 留痕 |
|---|---|---|---|
| 有人在 | 停下来问 | 用户确认 | 对话里可见 |
| 无人在（会话已卡、幽灵进程） | 直接执行 | 指纹、监视器信号 | 必须写审计记录，事后可查 |

**幽灵进程不能走"确认"那一路。** 幽灵进程的定义就是没人再管它，会话已经卡住，这时候不存在能点确认的人。所以它的处理只能是：自主收掉，然后留一条记录。

这不是例外处理，是同一套动作边界在两种在场状态下的两种形态。写文案时不要说成"一律先问"。

*English: a timeout without a terminal state leaves a session that looks busy forever. Ghost processes cannot be confirmed by a human who is not there: reap autonomously and leave an audit record.*

---

## 七、版本与依赖

`engines.dsh` 是一个 semver 范围，例如 `">=0.2.0 <0.3.0"`。**它有值时，安装器会完全忽略 peer 依赖**，所以这个字段要自己说清所有应当支持的版本。

预发布版本是个坑。当前 0.2.x 全是预发布，默认规则下：

| 写法 | 匹配 `0.2.1-rc.2` | 匹配 `0.2.1` 正式版 |
|---|---|---|
| `>=0.2.0 <0.3.0` | 否 | 是 |
| `>=0.2.0-rc.2 <0.3.0` | 否（0.2.1 的预发布） | 是 |
| `>=0.2.0-rc.2 <0.2.1 \|\| >=0.2.1-0 <0.3.0` | 是 | 是 |

只有分两段写，`includePrerelease` 取真或假时结果才一致。

*English: `engines.dsh` short-circuits peer checks. Prerelease versions need a two-cohort range or the result depends on how the range is evaluated.*

---

## 八、发请求的编码

把 JSON 当**字符串**交给 HTTP 客户端，会按 ISO-8859-1 编码。中文和长破折号会被吞掉，或者降级成 `??`——而且 HTTP 还是 200，接口还是返回了 id。

正确做法是序列化之后转成 UTF-8 字节数组，让字节数组当 body：

```js
const bytes = Buffer.from(JSON.stringify(payload), 'utf8');   // 用 bytes 发，不要用 string
```

发完必须**读回比对**，逐字符比。HTTP 200 且有 id，不等于存进去的字节是对的。

*English: send bytes, not strings. Then read the stored copy back and compare character by character.*

---
---

## 九、网络故障分三层看

<!-- wb-dsv4.1 2026-10-23T10:58 -->

现象都是「连不上」，但停在哪一层，处置完全不同。不分层，就会在错的地方反复改配置。

| 层 | 典型现象 | 判断依据 | 处置 |
|---|---|---|---|
| 代理层 | `CONNECT tunnel failed, response 502`，几乎立刻返回 | 代理端口没人监听，或代理拒绝这条隧道 | 换端口，或清空代理变量 |
| TCP 层 | `connect=0.000000`，等满 20 秒超时 | 握手都没完成 | 换通道，或重试 |
| 传输层 | connect 与 TLS 都很快（<0.3s），总时长 20s+，偶尔 000 | 前三段正常，卡在数据传输 | 原样重试一次 |

一条命令就能看出停在哪层：

```bash
curl -s -o /dev/null \
  -w "http=%{http_code} connect=%{time_connect} tls=%{time_appconnect} total=%{time_total}\n" \
  --max-time 40 https://github.com
```

`connect=0` 就是连握手都没成。`connect` 和 `tls` 都是零点几秒、而 `total` 二十多秒，就是传输层。

同一台机器上，`api.github.com` 和 `github.com` 的可达性可以完全不同。所以「我的网是不是坏了」这种问法没有意义，要问「哪个域名，停在哪一层」。

*English: three layers, three different fixes. Measure connect / TLS / total before changing any configuration.*

---

## 十、代理可能根本没在监听

环境变量里配了代理，不代表那个端口真的有东西在听。这一步不验，后面所有排查都是空的。

先看谁在听：

```bash
netstat -ano -p tcp | grep -i LISTENING | awk '{print $2}' | sed 's/.*://' | sort -n | uniq
```

再逐个试。用 api 域名快速筛，因为它通常最先通：

```bash
curl -s -o /dev/null -w "%{http_code}\n" --max-time 4 \
  -x http://127.0.0.1:<port> https://api.github.com/rate_limit
```

筛出候选之后，**必须再用 `github.com` 复筛一次**。两者可达性不一致是常态：有的代理只放行 api，不放行网页，拿 api 的结果当结论会选错。

`curl -x` 的退出码也能用：立刻返回（exit 7）是连接被拒，说明端口没人听；等满超时才叫被拦。

*English: a configured proxy is not a running proxy. Enumerate listeners, filter with the API host, then re-filter with the web host.*

---

## 十一、让代理只对 github 生效

改全局环境变量会影响所有程序。git 支持按 URL 配代理，只对指定域名生效：

```bash
git config --global http.https://github.com.proxy  http://127.0.0.1:<port>
git config --global https.https://github.com.proxy http://127.0.0.1:<port>
```

其他仓库照旧直连，不受影响。撤销：

```bash
git config --global --unset http.https://github.com.proxy
git config --global --unset https.https://github.com.proxy
```

一条实测限制：这类隧道代理会偶发返回 502，**失败就原样重试一次，不要改配置**。实测三次里成两次。这条与「传输层重试」是两回事，但处置相同。

*English: scope the proxy to one host. Expect occasional 502 from the tunnel and retry rather than reconfiguring.*

---

### 本节来源（wb-dsv4.1，2026-10-23）

上面三节由 wb-dsv4.1 于 2026-10-23 补写，来自当天的实测。

它修正的是本手册此前的结论「只有 github.com 的传输阶段会卡，处置就是重试，不用改任何配置」。
那句话在 2026-10-17 那次测量里成立，但不是规律：同一台机器在 10-23 出现了另外两层故障
（代理端口无人监听、TCP 握手失败），处置不同。**先分层，再决定要不要改配置。**

---

---

## 许可

MIT，见 `LICENSE`。

这里记的是做插件时踩过的坑，不是官方文档。接口细节以仓库源码为准，版本变了这里可能过时。
