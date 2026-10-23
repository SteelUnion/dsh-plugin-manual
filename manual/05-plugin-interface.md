# 第 5 册 · 插件接口层

写 DSH 插件时容易踩空的地方：钩子返回什么、为什么会停在半路、版本范围怎么填、请求怎么发。

---

## 5.1 先分清「没反应」是哪一种

「装了没效果」至少有四种，修法完全不同。不先分开，会一直在错的那层改代码。

| 现象 | 说明 | 先查什么 |
|---|---|---|
| 连模块都没加载 | 打包进去了，但没被 import | bundle 补丁里有没有 `- insert: - id/name` 这一项（空数组 `[]` 是什么都不注册） |
| 加载了，但 `apply()` 没跑 | 依赖没解析，或者配置校验抛错 | 顶层探针有输出、`apply()` 里的探针没有 |
| `apply()` 跑了，钩子没注册 | 注册时抛错，被 try 吞了 | 把注册的返回值打出来 |
| 钩子注册了，没被调用 | 事件没触发，或者被上游拦掉了 | 钩子入口第一行打探针 |

*English: four distinct failure modes, each with a different first check. Separate them before touching code.*

---

---

---

## 5.2 钩子返回什么

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

---

---

## 5.3 终态比超时重要

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

---

---

## 5.4 版本与依赖

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

---

---

## 5.5 发请求的编码

把 JSON 当**字符串**交给 HTTP 客户端，会按 ISO-8859-1 编码。中文和长破折号会被吞掉，或者降级成 `??`——而且 HTTP 还是 200，接口还是返回了 id。

正确做法是序列化之后转成 UTF-8 字节数组，让字节数组当 body：

```js
const bytes = Buffer.from(JSON.stringify(payload), 'utf8');   // 用 bytes 发，不要用 string
```

发完必须**读回比对**，逐字符比。HTTP 200 且有 id，不等于存进去的字节是对的。

*English: send bytes, not strings. Then read the stored copy back and compare character by character.*

---
---

---
