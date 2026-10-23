# 第 2 册 · 仓库与发布

往 GitHub 推东西：API 与 git 两条路怎么分、一次完整发布要走哪几步、出错时哪些动作是白费。

---

## 2.1 GitHub（`gh.ps1`）

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

---

## 2.2 上传到 GitHub（release.mjs）

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
1. 有没有**私人笔记**（草稿、todo、临时数据）
2. 有没有**数据集 / 截图 / 录屏**
3. 有没有**密钥、令牌、绝对路径**

发现私货文件 → `git rm --cached <file>`（**只取消跟踪，本地文件保留**），再写进 `.gitignore`。

```powershell
git rm --cached docs/personal-notes-zh.md
```

`.gitignore` 里加：

```gitignore
# Personal notes - kept locally, never published
docs/personal-notes-zh.md
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
