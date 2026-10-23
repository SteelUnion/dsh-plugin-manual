# DSH 3 · 插件读服务：注入门

> 适用面：**DSH 内核**。讲的是 DeepSeek Harness 的插件层：钩子语义、终态、注入门、
> 版本范围。换个宿主就不成立，别当头一条照抄。

> 插件不能直接 `ctx.shell`，也不能 `ctx.get()`。要么声明 `inject`，要么拿不到。

---

## 3.1 先声明 `inject`，再直接访问

**两种写法都会抛注入门错误**，`ctx.get()` 也不能绕：

```js
ctx.shell            // ✗ cannot get property "shell" without inject
ctx.get('shell')     // ✗ 同样抛
```

**正确做法：在插件里声明依赖。**

```js
export const inject = ['userQuestions'];   // 声明

export function apply(ctx) {
  ctx.userQuestions.ask(...);              // 声明后才能访问
}
```

`inject` 同时是**能力边界**：声明什么，插件才有什么。
中枢类插件只声明 `userQuestions`，不声明 `tools` / `shell`，就动不了手。

**这个错误只在真机暴露** —— 离线 mock 的 ctx 没有注入门，怎么写都不报错。
所以单测里要断言 `inject` 的内容：

```js
assert.ok(Array.isArray(mod.inject));
assert.ok(mod.inject.includes('userQuestions'));
assert.ok(!mod.inject.includes('tools'));   // 不该有的别声明
```
