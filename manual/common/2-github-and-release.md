# 第 2 册 · 仓库与发布

> 适用面：**通用**。命令、脚本、网络、调试方法，换个项目也照用。
> DSH 内核相关的内容不在这里，在 [`../dsh/`](../dsh/README.md)。

> 往 GitHub 推东西：API 与 git 两条路怎么分、出错时哪些动作是白费、实测基线与处置。
> 插件的打包与发版流程（`pack.mjs` → tag → Release → 附件）是 DSH 专有的，
> 已挪到 [`../dsh/4-package-and-release.md`](../dsh/4-package-and-release.md)。

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

## 2.3 发到 npm

Release 和 npm 是两条独立的发布通道，别混。Release 给人下载压缩包，npm 给 `npm i` 和
`dsh plugin add` 用。**只有 npm 有公开的下载计数**，想量「有没有人真的在用」，这是唯一能拿到的数字。

### 默认 registry 不能发

本机 `.npmrc` 多半指向 `registry.npmmirror.com`（镜像）。镜像是只读的，`npm publish` 会失败。
发布要显式指定官方源：

```
npm publish --registry=https://registry.npmjs.org/
```

### 先看清单，再发

```
npm pack --dry-run
```

输出里看三行：`Tarball Contents`（哪些文件进包）、`total files`、`package size`。
决定内容的是 `package.json` 的 `files` 字段。`files` 和 `.npmignore` 同时存在时容易互相打架，
只维护 `files`，别再写 `.npmignore`。

一个只有 `lib/index.js` 的插件，正常量级是 6 个文件、30 kB 上下。冒出几百 kB 通常是
`dist/` 或 `node_modules` 漏进去了。

### 认证

`npm publish` 需要 token。两种做法：

- 交互登录一次：`npm login`，会写进 `~/.npmrc`，明文 token，别提交进仓库
- 或者把 token 放进 CI 的 `NODE_AUTH_TOKEN` 环境变量

**token 不进仓库、不打印、不贴进对话。** 漏一次要立刻去 npm 后台 revoke。

### 有 scope 和没 scope 是两回事

`@owner/name` 里的 `owner` 必须是你自己的用户名或你所属的组织。没有那个 scope 发不出去，
而且报的是 `404 Not Found` 而不是权限错误，很容易误判成「名字被占了」。

查名字有没有被占，别看网页，看 registry：

```
curl -s -o /dev/null -w "%{http_code}" https://registry.npmjs.org/<包名>
```

`404` 是没被占，`200` 是已存在。

### 预发布版本要打 tag

版本号带 `-rc.1` / `-alpha.2` 时，`npm publish` **默认仍然会把它推成 `latest`**，
于是所有人装到的都是预览版。必须显式给 tag：

```
npm publish --tag next --registry=https://registry.npmjs.org/
```

正式版发出来之后 `latest` 才会回到正式版。发错了撤不回：72 小时内可以 `npm unpublish`，
超过就只能改版本号重发。
