# DSH 工具与插件手记

这份东西原本是内部速查，攒下来的都是「怎么做对」，不是「有哪些参数」。整理成公开版本时做了两件事：
把路径、人名、凭据相关的内容换成占位符；把只适用于某一台机器的细节标出来。

两块内容。前半讲在本机、在仓库里怎么把事做对：PowerShell、Python、Node、GitHub、发布流程、调试。
后半讲插件的接口层：钩子返回什么、为什么会停在半路、版本范围怎么填、请求怎么发。

内容以中文为主，每节末尾有一句英文摘要。接口细节以仓库源码为准，版本变了这里可能过时。

---

# 工具用法（正确写法合集）

> 只记「怎么做对」。按工具分类，可查可抄。

---

## 一、PowerShell

### Agent 工具 vs 手动终端

| | 版本 |
|---|---|
| Agent 侧的 `pwsh` 工具 | **PowerShell 7.6.6 Core** |
| 用户手动打开的终端 | **Windows PowerShell 5.1** |

让用户手跑的命令按 5.1 写；Agent 自己的工具按 7 写。

### 调用外部程序

```powershell
& "C:\path\to\prog.exe" -Flag "value"
```

`&` 开头。路径带空格必须加引号。

### here-string（多行文本、含 `$` 的模板）

```powershell
$q = @'
query ($searchQuery: String!, $first: Int!) {
  search(query: $searchQuery, type: DISCUSSION, first: $first) {
    discussionCount
  }
}
'@
```

规则：
- `@'` 单独占一行
- `'@` 顶格，前面无空格
- 内部 `$` 不展开，适合 GraphQL / bash / 任何模板

### 参数化脚本

```powershell
param(
  [string]$Q = 'default',
  [int]$Limit = 20,
  [switch]$Post
)
$ErrorActionPreference = 'Stop'
```

运行：

```powershell
& powershell -NoProfile -ExecutionPolicy Bypass -File .\script.ps1 -Q 'value' -Limit 10
```

`-NoProfile` 跳过用户配置，`-ExecutionPolicy Bypass` 免签名限制。

### 引用共享脚本

```powershell
. '<workspace>\gh.ps1'
```

`.` 开头是「点源」，在**当前作用域**加载，函数和变量都能直接用。

### JSON

```powershell
# 构造
$payload = [ordered]@{ query = $q; variables = [ordered]@{ x = 1 } }
$json = $payload | ConvertTo-Json -Depth 8 -Compress

# 读
$data = Get-Content 'x.json' -Raw -Encoding UTF8 | ConvertFrom-Json

# 写（无 BOM）
[System.IO.File]::WriteAllText($path, $text, [System.Text.UTF8Encoding]::new($false))
```

`-Depth` 默认是 2，嵌套深了会被截断。

### 抓全输出（含 stderr）

```powershell
$job = Start-Job { & cmd /c "程序 参数 2>&1" }
Start-Sleep -Seconds 14
$out = Receive-Job $job
Stop-Job $job -ErrorAction SilentlyContinue
Remove-Job $job -Force -ErrorAction SilentlyContinue
$out
```

用于启动类程序（GUI、server）。`2>&1` 把 stderr 并进 stdout。

### 限速

```powershell
$elapsed = ([datetime]::Now - $script:LastCall).TotalMilliseconds
if ($elapsed -lt 600) { Start-Sleep -Milliseconds ([int](600 - $elapsed)) }
$script:LastCall = [datetime]::Now
```

600ms = 1 秒 2 次。

### Git 提交中文信息

```powershell
git -c i18n.commitEncoding=utf-8 commit -q -F msgfile.txt
```

提交信息写文件，用 `-F` 传入。

### 查进程命令行

```powershell
Get-CimInstance Win32_Process -Filter "Name='DeepSeek Harness.exe'" |
  Select-Object ProcessId, CommandLine
```

---

## 二、文件工具（优先于 shell）

| 场景 | 工具 | 参数要点 |
|---|---|---|
| 读文件 | `read` | `file_path` + 可选 `offset`/`limit` |
| 找文件 | `glob` | `pattern` 如 `**/*.js`，`path` 指定目录 |
| 搜内容 | `grep` | `pattern` 正则，`include` 限文件类型 |
| 新建/整体重写 | `write` | 传完整内容 |
| 局部修改 | `edit` | `old_string` 必须**唯一命中** |

**改脚本/配置：用 `write` 整体重写，或 `edit` 精确替换。**
不用 shell 的字符串替换去改代码文件——会破坏里面的 `$变量`、引号、缩进。

**`edit` 前必须已读过该文件**（工具会强制）。

---

## 三、Python

### 解释器

```
<runtime>\python\python.exe
```

### 已装库

numpy · pandas · python-docx · python-pptx · openpyxl · Pillow · lxml · XlsxWriter

**未装**：yaml（要解析 YAML，用 Node 的 `js-yaml` 或手写）

### 调用

```powershell
& $py script.py arg1 arg2
& $py -c "import x; print(x.y)"
```

`-c` 里避开单引号嵌套，复杂逻辑写文件。

### 探针模板（未知二进制格式）

```python
import struct
with open(path, 'rb') as f:
    head = f.read(256)
for i in range(0, 64, 16):
    print(f'{i:04x}  {head[i:i+16]!r}')
```

先看 256 字节再决定怎么解析。

### asar 解包（`<workspace>\asar.py`）

```
python asar.py ls  <路径>
python asar.py x   <包名> ...
python asar.py find <关键词>
```

内部偏移：

```python
_, pickle_size, _json_len, data_len = struct.unpack('<4I', f.read(16))
f.seek(16)
J = json.loads(f.read(data_len).decode('utf-8'))
BASE = 8 + pickle_size        # 数据区起点
ROOT = {'files': J['files']}  # 根节点包一层
```

---

## 四、Node

### 解释器

```
<runtime>\node\node.exe
```

### 用法

```powershell
& $node --check file.js                    # 语法检查
& $node script.mjs                          # .mjs 直接跑
& $node --input-type=module -e "import('file:///E:/p/x.js').then(m=>console.log(Object.keys(m)))"
```

ESM 模块用 `file:///` 前缀的绝对路径 import。

### 写测试

`.mjs` 后缀、`import` 语法、末尾 `process.exit(0/1)` 给退出码。
假 ctx 模式：

```js
const handlers = {};
const ctx = { on(event, fn) { handlers[event] = fn; } };
apply(ctx, config);
await handlers['tools/pre-execute'](exec, next);
```

### zstd 压缩／解压（范例）

**参数一律用 `zlib.constants` 里的 `ZSTD_c_*` 符号：**

```js
import { zstdCompressSync, zstdDecompressSync, constants } from 'node:zlib';

// 压缩：带校验和（与 DSH 会话文件一致的写法）
const CHECKSUM_OPTIONS = { params: { [constants.ZSTD_c_checksumFlag]: 1 } };
const packed = zstdCompressSync(Buffer.from(text, 'utf8'), CHECKSUM_OPTIONS);

// 解压
const text = zstdDecompressSync(frame).toString('utf8');
```

### 按 zstd 帧切分（范例）

**追加式日志（如 `session.v4.jsonl.zstd`）是多帧拼接，帧边界就是索引。**

```js
const MAGIC = Buffer.from([0x28, 0xb5, 0x2f, 0xfd]);   // zstd frame magic

function splitFrames(buf) {
  const frames = [];
  let pos = 0;
  while (pos + 4 <= buf.length) {
    const idx = buf.indexOf(MAGIC, pos);
    if (idx === -1) break;
    const next = buf.indexOf(MAGIC, idx + 4);
    frames.push({ start: idx, end: next === -1 ? buf.length : next });
    pos = next === -1 ? buf.length : next;
  }
  return frames;
}
```

**局部重建时：坏的帧重压，其余帧 `buf.slice(f.start, f.end)` 原样拼回，帧数不变。**

---

## 五、GitHub（`gh.ps1`）

### 加载与调用

```powershell
. '<workspace>\gh.ps1'
$r = Invoke-Gh -Uri 'https://api.github.com/user'
$r = Invoke-Gh -Uri 'https://api.github.com/...' -Method POST -Body $json
```

内置 600ms 间隔（1 秒 2 次）。

### 凭据

存放在系统凭据管理器里，按条目标签取用；取出来是一个固定长度的访问令牌。
**令牌值不打印、不写入文件、不外传。**

### 配套脚本

```powershell
& powershell -NoProfile -ExecutionPolicy Bypass -File .\post_comment.ps1 -Number 9923 -File .\comment.json -Post
& powershell -NoProfile -ExecutionPolicy Bypass -File .\search_discussions.ps1 -Q "sandbox permission" -Limit 20
& powershell -NoProfile -ExecutionPolicy Bypass -File .\read_discussions.ps1
```

### GraphQL

```powershell
$queryText = @'
query ($a: String!) { ... }
'@
$payload = [ordered]@{ query = $queryText; variables = [ordered]@{ a = 'x' } }
$json = $payload | ConvertTo-Json -Depth 8 -Compress
$r = Invoke-Gh -Uri 'https://api.github.com/graphql' -Method POST -Body $json
```

**GraphQL 变量名用 `$searchQuery`/`$first`，PowerShell 参数用 `$Q`/`$Limit`** —— 两套名字别混。

### 常用端点

| 操作 | 方法与路径 |
|---|---|
| 建仓库 | `POST /user/repos` |
| 改仓库 | `PATCH /repos/{owner}/{repo}` |
| 设 topics | `PUT /repos/{owner}/{repo}/topics` |
| 建 release | `POST /repos/{owner}/{repo}/releases` |
| 回 discussion | GraphQL `addDiscussionComment` |
| 搜 discussion | GraphQL `search(type: DISCUSSION)` |

---

## 五之二、上传到 GitHub（`release.mjs`）

**位置**：`<workspace>\release.mjs`。在**任意 git 仓库里**调用，它自己找仓库根。

### 四个子命令

```powershell
$node = '<runtime>\node\node.exe'
$rel  = '<workspace>\release.mjs'

# 看一眼：仓库 / 分支 / HEAD / 工作区 / 与远端差多少
& $node $rel status

# 提交（自动 add -A），提交信息只从文件读
& $node $rel commit .\commitmsg.txt

# 只提交指定文件
& $node $rel commit .\commitmsg.txt -m README.md README.zh-CN.md

# 提交 + 推送一步到位（最常用）
& $node $rel ship .\commitmsg.txt

# 只推送
& $node $rel push [remote] [branch]
```

### 提交信息写法

放在一个 UTF-8 文件里，**第一行是标题（≤100 字符），空一行，后面是正文**。

```
docs: front-load the hard signals, add trade-offs and design rationale

The top of the README was narrative. A reader deciding in 30 seconds had to
scroll to the middle before seeing anything checkable.

- Add a What / Verified / Cost block directly under the title.
- Add "Trade-offs & known limitations".
```

脚本会自动处理：去 BOM、强制 UTF-8、标题超长报错、空信息报错、没改动就跳过。

### 标准流程

```powershell
Set-Location '<repo-dir>'
& $node $rel status              # 1. 先看工作区和远端
# ... 改文件 ...
& $node $rel ship .\commitmsg.txt  # 2. 提交并推送
& $node $rel status              # 3. 复查是否一致
```

**第 3 步不能省**。`ship` 报错退出不代表提交没成功 —— `commit` 和 `push` 是两步，
push 失败时提交已经落地，重跑一次 `push` 即可，不要重跑 `ship`。

### GitHub 推送：实测基线与超时处置

**基线（2026-10-17 本机实测，来源：本机连接诊断记录）**

| 域名 | connect | TLS | total | 结果 |
|---|---|---|---|---|
| `api.github.com` | 0.110s | 0.216s | **0.317s** | 200 |
| `raw.githubusercontent.com` | 0.078s | 1.539s | **1.765s** | 200 |
| `github.com` | 0.136s | 0.269s | **23.75s** | 200 |
| `github.com` | 0.133s | 0.265s | **20.71s** | 000（连接重置） |
| `github.com` | 0.133s | 0.257s | **19.62s** | 000（连接重置） |

**读法：DNS / TCP / TLS 三段全部正常，卡在第四段「数据传输」。**
而且是**会断**，不只是慢——三次里两次直接 `000`。

**正确处置：直连重试一次。**

```powershell
git push origin main:main
```

实测有效：2026-10-18 推 `confirm-gate` 的 README，`ship` 第一次
`ssh: connect to host ssh.github.com port 443: Connection timed out`，
原样重跑一次 `git push` 即成功。**不用改任何配置。**

**三个别试的方向（都有实测支撑）**

| 方案 | 为什么无效 |
|---|---|
| 换 DNS | 系统 DNS / `223.5.5.5` / `8.8.8.8` 三个解析器给出**同一个 IP** `20.205.243.166`，是 GitHub 官方亚太节点，没被污染 |
| 改 hosts 换 IP | 试过的 IP 直接 `000` 连不上 |
| 加速中转站 | **只管 `raw.`，管不了 git 协议**；且直连 529ms 比所有中转都快（gh-proxy 1293ms / ghproxy.net 1538ms / ghfast.top 403 / mirror.ghproxy.com 超时） |

**判断依据**：看是哪个域名。`api.github.com` 和 `raw.githubusercontent.com` 都正常，
**只有 `github.com` 这一个域名的传输阶段会卡**。所以走 API 的操作（`gh.mjs` 建仓库、
建 release、回 discussion）不受影响，只有 `git push` / `git fetch` / `git clone` 会撞上。

**连续 3 次仍超时**，再按「三次上限」停下来汇报，不要自己找偏方。

---

### 发一个完整版本（改代码 → 打包 → 推送 → tag → Release → 传附件）

**六步，按顺序做。** 以 `dsh-confirm-gate` v0.1.2 为例。

#### 第 0 步：先看仓库里有什么，别把私货传上去

```powershell
Set-Location '<repo-dir>'
git ls-files                     # 已跟踪
git status --porcelain --ignored # 未跟踪 + 被忽略
```

**提交前必须看清名单。** 检查三件事：
1. 有没有**私人笔记**（面试速查、todo、草稿）
2. 有没有**数据集 / 截图 / 录屏**
3. 有没有**密钥、令牌、绝对路径**

发现私货文件 → `git rm --cached <file>`（**只取消跟踪，本地文件保留**），再写进 `.gitignore`。

```powershell
git rm --cached docs/interview-notes-zh.md
```

`.gitignore` 里加：

```gitignore
# Personal notes — kept locally, never published
docs/interview-notes-zh.md
```

**验证**：`git status --porcelain --ignored` 里该文件应该显示 `!!`（被忽略），而不是 `??`（未跟踪）。

#### 第 1 步：改代码，跑测试

```powershell
$node = '<runtime>\node\node.exe'
& $node test/integration.mjs
# 期望最后一行：===== ALL PASS (27 checks) =====
```

**测试不过就别往下走。**

#### 第 2 步：改版本号

两处一起改，否则打出来的包名和包内版本对不上（`pack.mjs` 会报错拦下）：

| 文件 | 字段 |
|---|---|
| `package.json` | `"version": "0.1.2"` |
| `README.md` / `README.zh-CN.md` | 头部那行的 `**v0.1.2**` |
| `dist/README.md` | 「当前版本」那一行 |

**顺带看 peer 依赖**：写成具体版本（`"0.2.0-rc.2"`）会在新版本 DSH 上报警告。
用范围 `">=0.2.0-rc.2"`，并在 README 里声明**验证于哪个版本**。

#### 第 3 步：打包

```powershell
& $node pack.mjs
```

输出到 `dist/dsh-confirm-gate-v<version>.zip`。脚本从 `package.json` 读版本，
白名单收文件（`INCLUDE` 数组，不用通配），**同样的输入打出同样的字节**（时间戳固定）。

#### 第 4 步：提交并推送

```powershell
& $node $rel status            # 先看
& $node $rel commit .\commitmsg.txt
git push origin main:main      # 超时就原样重跑一次
```

#### 第 5 步：打 tag

```powershell
git tag -a v0.1.2 -m "v0.1.2 - <一句话>"
git push origin v0.1.2
```

tag 要**单独 push**（`git push origin main` 不带 tag）。

#### 第 6 步：建 Release + 传附件

**建 Release** —— 请求体写 JSON 文件，用 `gh.mjs` 发：

```json
{
  "tag_name": "v0.1.2",
  "name": "v0.1.2 — works on 0.2.1-rc.2 and later",
  "body": "markdown 正文",
  "draft": false,
  "prerelease": false
}
```

```powershell
$gh = '<workspace>\gh.mjs'
& $node $gh post /repos/<owner>/<repo>/releases --body-file .\release_body.json
```

**记下返回里的 `id`**（例：`416344000`）。

**传附件** —— 用 `upload_asset.mjs`，三个参数：

```powershell
$ua = '<workspace>\upload_asset.mjs'
& $node $ua 'dist\dsh-confirm-gate-v0.1.2.zip' <release-id> <owner>/<repo>
```

#### 第 7 步：验证

```powershell
& $node $gh get /repos/<owner>/<repo>/releases/latest
```

看三样：`tag_name` 对不对、`name` 对不对、`browser_download_url` 和 `size` 有没有。

**别跳过这一步** —— Release 建成功但附件没传上去，界面看起来是正常的。

### 完整键位速查

| 动作 | 命令 |
|---|---|
| 看仓库状态 | `node release.mjs status` |
| 提交（信息从文件读） | `node release.mjs commit msg.txt` |
| 提交 + 推送 | `node release.mjs ship msg.txt` |
| 只推送 | `git push origin main:main` |
| 打 tag | `git tag -a vX.Y.Z -m "..."` |
| 推 tag | `git push origin vX.Y.Z` |
| 建 Release | `node gh.mjs post /repos/o/r/releases --body-file body.json` |
| 传附件 | `node upload_asset.mjs file.zip <release-id> o/r` |
| 查最新 Release | `node gh.mjs get /repos/o/r/releases/latest` |
| 打包 | `node pack.mjs` |
| 取消跟踪（保留本地） | `git rm --cached <file>` |

---

## 六、调试

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

## 七、写之前过一遍

- [ ] 我要处理的数据格式，**我见过真实样本**吗？没见过 → 先探针
- [ ] 这段跑在 **7 还是 5.1**？
- [ ] 有 `$变量` 会被 PowerShell 抢先展开吗？→ 单引号 here-string
- [ ] 改文件用 `write`/`edit`，还是字符串替换？
- [ ] 输出长度可控吗？→ 加 `Select-Object` / `-First`
- [ ] 失败时能看到原因吗？→ `2>&1` + 显式输出

---

---

## 后半部分：插件接口层

<!-- wb-dsv4.1 2026-10-23T11:05 -->

从第八节开始由 wb-dsv4.1 整理，内容是写 DSH 插件时容易踩空的地方：钩子注册了不触发、
等待没有终态、版本范围填错、请求发出去字节被改掉。第十三到十五节是 2026-10-23 补写的网络分诊，
修正的是「只有传输阶段会卡」这个只在一个测量窗口内成立的结论。

前七节来自原有内部速查，路径、人名、凭据相关内容已做占位化处理。

---

## 八、先分清「没反应」是哪一种

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

## 九、钩子返回什么

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

## 十、终态比超时重要

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

## 十一、版本与依赖

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

## 十二、发请求的编码

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

## 十三、网络故障分三层看

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

---

## 十四、代理可能根本没在监听

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

---

## 十五、让代理只对 github 生效

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

---

## 许可

MIT，见 `LICENSE`。

这份手记有一部分由内部速查整理而来。路径与人名已做占位化处理（`<runtime>`、`<workspace>`、
`<repo-dir>`），落到自己的环境时按实际替换。
