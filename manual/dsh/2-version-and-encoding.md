# DSH 2 · 版本范围与请求编码

> 适用面：**DSH 内核**。讲的是 DeepSeek Harness 的插件层：钩子语义、终态、注入门、
> 版本范围。换个宿主就不成立，别当头一条照抄。

> `engines.dsh` 怎么填才不会被预发布版本坑；以及请求字节在哪一层被改掉。

---

## 2.1 版本与依赖

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

## 2.2 发请求的编码

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
