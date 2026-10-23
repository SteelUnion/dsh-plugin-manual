# DSH 4 · 插件打包与发版

> 适用面：**DSH 内核**。讲的是 DeepSeek Harness 的插件层：钩子语义、终态、注入门、
> 版本范围。换个宿主就不成立，别当头一条照抄。

> 一个插件从改代码到 Release 附件，六步。以 `dsh-confirm-gate` 为例。

---

## 4.1 发一个完整版本（改代码 → 打包 → 推送 → tag → Release → 传附件）

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
