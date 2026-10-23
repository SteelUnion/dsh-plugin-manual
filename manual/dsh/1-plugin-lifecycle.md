# DSH 1 · 插件生命周期

> 适用面：**DSH 内核**。讲的是 DeepSeek Harness 的插件层：钩子语义、终态、注入门、
> 版本范围。换个宿主就不成立，别当头一条照抄。

> 插件装上但没效果时按这三节往下切：先分清「没反应」是哪一种，再看钩子返回什么，
> 最后看它为什么停在半路而没有终态。

---

## 1.1 先分清「没反应」是哪一种

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

## 1.2 钩子返回什么

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

## 1.3 终态比超时重要

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
