# 第 1 册 · 工具与运行时

在这台机器上把命令跑对：shell 版本差异、文件工具、Python 与 Node 解释器、几个现成脚本。

---

## 1.1 PowerShell

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

---

## 1.2 文件工具（优先于 shell）

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

---

## 1.3 Python

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

---

## 1.4 Node

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
