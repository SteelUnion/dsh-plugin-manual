# 第 3 册 · 网络分诊

> 适用面：**通用**。命令、脚本、网络、调试方法，换个项目也照用。
> DSH 内核相关的内容不在这里，在 [`../dsh/`](../dsh/README.md)。

> 连不上的时候按层查：代理层、TCP 层、传输层，现象和处置都不一样，先分层再动手。

---

## 3.1 网络故障分三层看

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

---

## 3.2 代理可能根本没在监听

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

---

## 3.3 让代理只对 github 生效

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
