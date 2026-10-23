# 第 4 册 · 调试与自检

东西不工作时怎么定位：探针放哪、分层问什么、写之前先过一遍什么。

---

## 4.1 调试

### 探针优先

不确定一段代码有没有执行 → **在它里面写一行文件**，看落不落盘。

```js
import { appendFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
const PROBE = join(tmpdir(), 'probe.log');
function probe(m) { try { appendFileSync(PROBE, `${new Date().toISOString()} ${m}\n`); } catch {} }
```

### 模块级探针（区分「没加载」和「加载了但没生效」）

顶层写一行，比 `apply()` 更早：

```js
probe(`MODULE LOADED pid=${process.pid} cwd=${process.cwd()} file=${import.meta.url}`);
```

| 日志 | 结论 |
|---|---|
| 连这条都没有 | 模块**没被 import** → 查加载配置 |
| 有这条，没 `apply()` | 加载了但没初始化 → 查返回值/依赖 |
| 有 `apply()`，钩子没触发 | 钩子没挂上 → 查注册 |
| 钩子触发但行为不对 | 逻辑问题 → 看钩子内探针 |

### DSH 插件读服务：先声明 `inject`，再直接访问

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

### 失效方向：宁可多报警，不可静默

读不到状态时**倾向照常工作**，不要保守沉默。

| 失效方式 | 代价 |
|---|---|
| 该沉默时报警 | 多一条提醒，**看得见、改得掉** |
| 该报警时沉默 | **插件装着却什么都不做**，日志看不出异常 |

对记账/提醒类工具，后者严重得多。安全边界才该 fail-closed。

### 判断运行时状态

系统提示 / 配置文件里的值是**快照**。要确认当前真实状态：
- 查进程命令行
- 或实测行为

### 分层验证

一个功能不工作时，逐层问：
1. 文件在不在？（`Test-Path`）
2. 模块加载了吗？（顶层探针）
3. 初始化跑了吗？（`apply()` 内探针）
4. 钩子挂上了吗？（注册返回值）
5. 钩子被调用了吗？（钩子内探针）

哪一层没打出来，问题就在那一层。

---

---

## 4.2 写之前过一遍

- [ ] 我要处理的数据格式，**我见过真实样本**吗？没见过 → 先探针
- [ ] 这段跑在 **7 还是 5.1**？
- [ ] 有 `$变量` 会被 PowerShell 抢先展开吗？→ 单引号 here-string
- [ ] 改文件用 `write`/`edit`，还是字符串替换？
- [ ] 输出长度可控吗？→ 加 `Select-Object` / `-First`
- [ ] 失败时能看到原因吗？→ `2>&1` + 显式输出

---

---
