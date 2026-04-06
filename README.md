![img](./logo.png)

<div align="center">

  <h1>Total ECH</h1>
  
  <p><b>为 CDN 全面开启 ECH 的 DoH 服务</b></p>
  <p>基于 Cloudflare Workers，适用于 Cloudflare CDN / Meta CDN</p>
</div>

![img](./mascot.png)

<div align="center">
  <p><b>注入满满的 ECH 能量！</b></p>
</div>

---

## ❓ ECH 是什么？

**ECH (Encrypted Client Hello，加密客户端问候)** 是一种 TLS 协议扩展，用于增强网络隐私。它加密 TLS 握手中的 Client Hello 明文信息（特别是 SNI），防止网络运营商或攻击者窥探用户访问的具体网站。它是 ESNI 的继任者，通过 DoH (DNS over HTTPS) 获取公钥来实现更安全的网站访问。

---

## ✨ 核心效果

| 功能效果 | 详细说明 |
|---|---|
| **1. 保护您的隐私，对ISP隐藏真实访问目标** | 强制开启ECH：<br>• **Cloudflare CDN**：所有 CF 网站（包括 X, Discord, Pixiv 等）<br>• **Meta CDN**：Facebook, Instagram, Threads, WhatsApp 等<br>可让您的**ISP**只能看到您在访问 Cloudflare / Meta CDN，无法确切知道您访问的具体网站或服务。|
| **2. 对代理提供商隐藏真实访问目标** | 在全局代理的情况下后置使用 ECH，可让您的**代理提供商**只能看到您在访问 Cloudflare / Meta CDN，无法确切知道您访问的具体网站或服务。 |
| **3. 自定义优选 IP** | 能够将默认的 Cloudflare IP 替换为您自己测试出的优选 IP。 |

> [!CAUTION]
> **安全风险警示**：如果**直连**，则目标网站可得知您的真实 IP，且无法绕过地区限制，请慎重使用！全局代理后置使用 ECH 则无此风险。

---

## ⚙️ 原理

| 平台 | 实现机制 | 轮换策略 |
|---|---|---|
| **Cloudflare CDN** | 边缘采用相同的 ECH 配置。通过查询开启 ECH 的 CF 域名的 HTTPS 记录，获取通用 ECH 配置。（注：CF 面板的 ECH 开关仅影响 HTTPS 记录是否包含 ECH 配置） | 每 1 小时轮换 1 次，有效时间 > 3 小时。 |
| **Meta CDN** | 边缘采用通用配置策略，涵盖几乎所有 Meta 服务。目前为内部测试，未通过 DNS 发布且暂无轮换。通过 `Hello Retry Request` 获取。 | 暂无轮换（内测阶段）。 |
| **本项目机制** | IP 为 Cloudflare/Meta CDN 的域名，在解析 HTTPS 记录时，**自动增加**各自的通用 ECH 配置。 | 自动处理。 |

---

## 🚀 部署方式

登录 Cloudflare 控制台，进入 Workers 页面并新建一个 Worker 。

复制[_worker.js](https://github.com/RememberOurPromise/Total-ECH/blob/main/_worker.js)代码粘贴到在线编辑器中，在此您可以自定义path。

点击“保存并部署”即可完成手动部署。

详细步骤请参考标准的 Cloudflare Workers 项目部署方式。

部署完成后，**请绑定您自己的域名**。

---

## 📖 使用方法

### 1. DoH 接口与参数配置

以下是本项目提供的 DoH 接口及优选 IP 替换参数说明：

| 接口类型 | URL 格式 / 参数 | 说明与示例 |
|---|---|---|
| **标准 ECH DoH** | `https://example.com/doh-ech-test` | 添加 ECH 但不进行优选 IP 替换的 DoH。 |
| **附加优选域名** | `?cf=[优选域名]` | 示例：`https://example.com/doh-ech-test?cf=youxuan.example.com`<br>*(注：必须填写真正返回 CF IP 的域名)* |
| **附加优选 IPv4** | `?ip4=[IPv4地址]` | 示例：`https://example.com/doh-ech-test?ip4=104.18.11.118` |
| **附加优选 IPv6** | `?ip6=[IPv6地址]` | 示例：`https://example.com/doh-ech-test?ip6=2606:4700::6812:a76` |
| **附加优选 IPv4+IPv6** | `?ip4=[IPv4地址]&ip6=[IPv6地址]` | 示例：`https://example.com/doh-ech-test?ip4=104.18.11.118&ip6=2606:4700::6812:a76` |
| **纯净转发 DoH** | `https://example.com/doh-test` | 仅转发，DNS记录不作任何修改（直接转发 Google DoH）。 |

**特殊域名解析：**  `cf.ech`  解析其 HTTPS 记录会返回 CF 通用 HTTPS 记录 (2-3小时 TTL)，**适合 Xray 使用**。 

> [!NOTE]
> **优先级提示**：如果您同时填写了优选 IP (`ip4`/`ip6`) 和优选域名 (`cf`)，**优选 IP 的优先级高于优选域名**。

---

### 2. 浏览器防降级配置 (强制等待 HTTPS 记录)

配置浏览器的 DoH 服务后（请自行查阅 Chromium/Firefox 的基础 DoH 设置），为了防止 HTTPS 记录返回较慢时 ECH 回退，需要强制浏览器等候 HTTPS 记录。

### **Chromium:**

添加启动参数:
`--enable-features="UseDnsHttpsSvcb:UseDnsHttpsSvcbEnforceSecureResponse/true"`

说明:
>  Param to control whether or not HostResolver, when using Secure DNS, will fail the entire connection attempt when receiving an inconclusive response to an HTTPS query (anything except transport error, timeout, or SERVFAIL). Used to prevent certain downgrade attacks against ECH behavior.
>  此参数控制 HostResolver 在使用安全 DNS 时，如果收到 HTTPS 查询的不确定响应（传输错误、超时或 SERVFAIL 以外的任何响应），是否会终止整个连接尝试。用于防止针对 ECH 行为的某些降级攻击。

**Chromium 快捷方式启动示例：**

新建一个快捷方式，右键，属性，目标后面加上参数
```
C:\Users\XXXXXXX\AppData\Local\Google\Chrome\Application\chrome.exe --enable-features="UseDnsHttpsSvcb:UseDnsHttpsSvcbEnforceSecureResponse/true"
```
保存，结束进程重新启动

**Chromium 脚本启动示例：**
```
:: 批处理脚本 (start.bat) 示例
START "" "C:\Users\XXXXXXX\AppData\Local\Google\Chrome\Application\chrome.exe" --enable-features="UseDnsHttpsSvcb:UseDnsHttpsSvcbEnforceSecureResponse/true"
```

### **Firefox：**

地址栏输入 `about:config`
修改配置

| 配置项  | 设定值 | 作用说明 |
|---|---|---|
| `network.dns.force_waiting_https_rr` | `true` | 发起连接前强制等待 HTTPS 记录查询结果。 |
| `network.dns.echconfig.fallback_to_origin_when_all_failed` | `false` | 当所有 ECH / HTTPS 记录均验证失败或未拿到时，是否允许回退到不使用 ECH 的原始明文 SNI 连接。|


> [!WARNING]
> 🛑 **重要避坑指南**：Chromium / Firefox 在使用**系统代理**或 **Socks5 代理**时，会**关闭 DoH** 并采用远程解析，这将导致 ECH 完全失效！

---

## 🙏 致谢

感谢以下组织、项目与作者提供的灵感与基础：

*   **IETF**: [RFC9849](https://datatracker.ietf.org/doc/rfc9849/) (始作俑者)
*   [**Cloudflare, Inc.**](https://www.cloudflare.com/) (赛博菩萨)
*   **[Project X](https://github.com/XTLS/) / [Xray-core](https://github.com/XTLS/Xray-core/)**: [@RPRX](https://github.com/rprx), [@Fangliding](https://github.com/Fangliding), [@yuhan6665](https://github.com/yuhan6665) (idea以及Xray-core实现ECH配置查询和实际访问域名的解耦)
*   [**Towards a Complete View of Encrypted Client Hello Deployments**](https://dl.acm.org/doi/pdf/10.1145/3744969.3748401) (作者: JONAS MÜCKE, KONSTANTIN GASSER, THOMAS C. SCHMIDT, MATTHIAS WÄHLISCH) (Meta CDN的ECH部署情况以及HRR探测方式)
*   [**OpenSSL**](https://github.com/openssl/openssl) (OpenSSL 4.0 正式引入对ECH支持)
*   [**Meta Platforms, Inc.**](https://www.meta.com/) (很遗憾我不用Meta系的服务，XD)
---

## ⚠️ 免责声明

*   本项目是娱乐项目，Vibe Coding 且仅为 PoC 性质，不作任何保证！
*   本项目可能会导致某些不可访问的网站可以访问，这仅仅是意外！
*   本项目仅处理A AAAA HTTPS记录且仅测试了桌面版Chrome和Firefox！
*   本项目与 Cloudflare, Meta 没有任何关系！
*   本项目是 Gemini 3 写的，OpenClaw 发布的！没有任何人类参与，所以，水表电表都在门外最近没有点外卖快递都是自提请勿上门！！

---

## 📄 许可证

**无版权！**

大家玩的开心就好！

---

## Demo

`https://img1.eu.cc/doh-ech-test?&ip4=104.18.11.118&ip6=2606:4700::6812:a76`

`https://api1.eu.cc/doh-ech-test?&ip4=104.18.11.118&ip6=2606:4700::6812:a76`

`https://cdn2.eu.cc/doh-ech-test?&ip4=104.18.11.118&ip6=2606:4700::6812:a76`

每个每天10w请求，CF Workers免费额度

`eu.cc`是免费二级域名

不记录日志不维护，自然死

---

[1](https://github.com/net4people/bbs/issues/529)
[2](https://github.com/XTLS/Xray-core/discussions/5199)
[3](https://github.com/XTLS/BBS/issues/1)
[4](https://github.com/XTLS/BBS/issues/13)
