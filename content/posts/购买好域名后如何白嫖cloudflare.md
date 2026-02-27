---
title: "购买好域名后如何白嫖 Cloudflare：从注册到完整配置实战"
date: 2026-02-26
lastmod: 2026-02-27
description: "手把手教你在购买好域名后，用 0 成本接入 Cloudflare，完成 DNS、SSL、CDN、邮箱等关键配置。"
categories: ["建站实战"]
tags: ["Cloudflare", "DNS", "SSL", "免费建站", "域名配置"]
keywords: ["Cloudflare DNS 配置", "Cloudflare SSL", "Cloudflare Email Routing", "域名接入 Cloudflare", "NameSilo 教程"]
ShowToc: true
faq:
  - q: "域名从 NameSilo 接入 Cloudflare 后，多久生效？"
    a: "通常几分钟到数小时，极端情况可达 24 小时，生效后 Cloudflare 面板会显示 Active。"
  - q: "Cloudflare 免费版够个人博客使用吗？"
    a: "够用。DNS、SSL、CDN、Pages 托管和基础安全能力都可满足个人技术博客的日常需求。"
draft: false
---

## 一、前言：为什么一定要用 Cloudflare？

大多数人买完域名，只停留在“能访问就行”。  
但对开发工程师/内容创作者来说，更理想的状态是：

- **全站 HTTPS 加密**，访问更安全、权重更友好；
- 有一层 **全球 CDN 加速和 DDoS 防护**；
- 可以方便地绑定各种服务（Pages、Workers、邮箱、对象存储等）；
- 最好是 **完全免费、不自己运维服务器**。

Cloudflare 的免费套餐正好满足这些需求。  
这篇文章就是：**从你已经买好域名开始，教你如何“白嫖” Cloudflare 的核心能力，并把域名配置完整。**

> 本文示例以 `maquren.top` 为例，域名注册商以 NameSilo 为例，其他注册商（阿里云、腾讯云等）操作类似。

---

## 二、前置条件：你需要准备什么

开始之前，请确认你已经具备：

- **一个已购买的域名**（例如：`maquren.top`），能登录到注册商后台（如 NameSilo）；
- 一个 **Cloudflare 账号**，如果没有可以先免费注册；
- 能访问 Cloudflare 控制台（`https://dash.cloudflare.com`）；
- 对 DNS 解析有基础概念（知道 A 记录 / CNAME 的大致含义即可）。

如果你还没有 Cloudflare 账号：

1. 打开 `https://dash.cloudflare.com`；
2. 使用邮箱注册一个免费账号（Free Plan 就够用）；
3. 登录后即可开始添加站点。

---

## 三、第一步：在 Cloudflare 添加你的域名

1. 登录 Cloudflare 控制台；
2. 点击右上角的 **“Add a site” / “添加站点”**；
3. 输入你的域名（例如 `maquren.top`），注意不要带 `http://` 或子域名；
4. 选择套餐时，直接选 **Free（免费版）**；
5. 继续后，Cloudflare 会尝试扫描你当前的 DNS 记录：
   - 如果是新域名，基本上没有什么记录；
   - 如果之前在别的地方已经有解析，可以先保留这些记录，后面再精简。

**建议：**

- 不要一上来就删掉所有旧记录，先记下原解析情况再决定；
- 对于完全新域名，可以暂时只保留几条必要记录（后面会讲）。

---

## 四、第二步：在注册商处更换 DNS（NameSilo 为例）

Cloudflare 要工作，需要成为你的域名的 **权威 DNS**。  
这一步就是在注册商后台，把原来的 NS 记录改成 Cloudflare 提供的 NS。

1. 在 Cloudflare 完成“添加站点”步骤后，会看到类似这样的提示：

```text
Please change your nameservers to:
  xxxx.ns.cloudflare.com
  yyyy.ns.cloudflare.com
```

2. 登录 **NameSilo**（或你的域名注册商），找到你的 `maquren.top`：
   - 一般在 “Domain Manager / 域名管理”；
3. 找到 **Nameservers / 域名服务器** 设置；
4. 将原来的 NS 修改为 Cloudflare 给你的那两个：

```text
ns1: xxxx.ns.cloudflare.com
ns2: yyyy.ns.cloudflare.com
```

5. 保存修改。

> 注意：不同域名注册商界面不同，但关键词大多是 “Nameserver / DNS / 域名服务器”。

修改完成后，需要等待 **几分钟到 24 小时** 的生效时间。  
Cloudflare 控制台会自动检测，当看到状态变为 **“Active”** 或显示绿色对勾时，说明 NS 已经切换成功。

---

## 五、第三步：整理和配置 DNS 解析

接下来是在 Cloudflare 里配置你的 DNS 记录，这一步决定：

- 域名解析到哪里；
- 是否走 Cloudflare CDN / WAF；
- 子域名如何划分。

进入 Cloudflare → 选择你的域名 → 左侧点击 **DNS** → **Records**。

### 1. 基本概念回顾

- **A 记录**：指向一个 IPv4 地址（例如某台服务器）。
- **CNAME 记录**：指向另一个域名，通常用来接到 Pages/其他 SaaS 服务。
- 云朵图标：
  - **橙色**：走 Cloudflare 代理（CDN/加速/防护）；
  - **灰色**：只做 DNS 解析，不走 Cloudflare。

### 2. 为根域名（`maquren.top`）配置解析

根据你后续要用的服务不同，这里有几种常见情况。

#### 场景 A：暂时没有服务器，只想先有一个占位页

可以先指向一个占位 IP（比如后面会用到的服务器），或者暂时留空，等你接入 Cloudflare Pages 再设置 CNAME。

**推荐做法（配合 Cloudflare Pages）：**

先暂时不配置根域名的 A 记录，只保留 Cloudflare 自动添加的默认记录。  
当你在 Cloudflare Pages 中创建项目，并绑定自定义域时，Pages 会指导你添加一条 **CNAME 记录**，例如：

```text
类型：CNAME
名称：maquren.top
目标：your-project-name.pages.dev
代理：橙色云朵（开启）
```

> 这一部分可以在你真正创建完 Pages 项目后再回来补，不一定要第一时间完成。

#### 场景 B：有自己的服务器（例如一台国内/海外 VPS）

你可以添加一条 A 记录：

```text
类型：A
名称：@
IPv4：你的服务器 IP
代理：建议橙色云朵（隐藏真实 IP，享受 CDN）
```

这里的 `@` 代表根域名 `maquren.top`。

### 3. 为子域名（如 `www.maquren.top`）配置解析

一般建议：

- `www.maquren.top` → 指向根域名或 Pages；

示例：

```text
类型：CNAME
名称：www
目标：maquren.top
代理：橙色云朵
```

或者：

```text
类型：CNAME
名称：www
目标：your-project-name.pages.dev
代理：橙色云朵
```

### 4. 检查多余记录

Cloudflare 导入时可能会带入一些不必要的记录（例如原来免费邮箱、停车页等）：

- 如果你确定不用，可以删除这些多余的 A/CNAME/MX 记录；
- 保持 DNS 列表干净，有助于排错和维护。

---

## 六、第四步：配置 SSL/TLS，开启全站 HTTPS

在 Cloudflare 控制台 → 选择你的域名 → 左侧点击 **SSL/TLS**。

### 1. 选择加密模式

在 **Overview / 概览** 里有一个 “Encryption mode / 加密模式”：

- 如果你的源站（比如 Pages、支持 HTTPS 的服务器）本身支持 HTTPS：
  - 建议选择 **Full** 或 **Full (strict)**；
- 如果你暂时没有 HTTPS 源站，只是想给用户端到 Cloudflare 之间加密：
  - 可以先用 **Flexible**（灵活），后面再升级。

对于使用 **Cloudflare Pages** 的场景，可以直接用 **Full (strict)**。

### 2. 强制 HTTP 自动跳转到 HTTPS

在 **SSL/TLS → Edge Certificates** 里：

- 开启 **“Always Use HTTPS”**；
- 可选：开启 **“Automatic HTTPS Rewrites”**；

这样，无论访问者输入 `http://maquren.top` 还是 `https://`，最终都会跳到 HTTPS。

---

## 七、第五步：配置 Cloudflare Email Routing（可选但强烈推荐）

如果你想要一个 **专业感更强的邮箱地址**，例如：

```text
contact@maquren.top
hi@maquren.top
```

但又不想自己搭邮局服务器，可以使用 **Cloudflare Email Routing**（邮件转发）。

1. 在 Cloudflare 控制台中选择你的域名；
2. 左侧找到 **Email → Email Routing**；
3. 按向导开启 Email Routing；
4. 设置一个或多个自定义邮箱地址，并指定转发到你的实际邮箱（如 Gmail / QQ 邮箱）；
5. Cloudflare 会自动为你添加所需的 MX 记录和 TXT 记录。

> 这样，对外你可以用 `xxx@maquren.top` 作为收件邮箱，实际邮件会被转发到你常用的邮箱，非常适合对外展示和业务联系。

---

## 八、第六步：基础性能与安全优化

Cloudflare 免费版已经有不少默认不错的设置，你可以按需做一些增强。

### 1. 打开基础缓存与压缩

在 **Speed / 性能** 和 **Caching / 缓存** 菜单下：

- 确认启用了：
  - 静态资源缓存；
  - `gzip` / `brotli` 压缩；
- 一般默认就不错，新手不需要太多调整。

### 2. 开启基础安全规则（WAF）

在 **Security / 安全 → WAF** 中：

- 免费版会有一些基础规则；
- 可以启用常见的防护规则，比如通用 OWASP 规则；
- 对于新站点，保持默认 + 观察即可，不用一开始就很激进。

---

## 九、第七步：绑定 Cloudflare Pages（如果你要做静态博客/网站）

如果你打算用 Cloudflare Pages 托管网站（比如 Hugo 博客），可以这样做：

1. 在 Cloudflare → Pages 创建项目，连接 GitHub 仓库；
2. 配置构建命令（例如 `hugo`）和输出目录（例如 `public`）；
3. 首次构建后，Pages 会给你一个 `xxx.pages.dev` 的地址；
4. 在 Pages 的 “自定义域” 中添加：  
   - `maquren.top` 或 `www.maquren.top`；
5. 按提示，在 DNS 中添加对应的 CNAME 记录（通常会自动帮你添加）。

完成后，你就可以通过自己的域名访问由 Pages 托管的静态网站了。

> 更详细的 Pages 搭建过程，可以结合《域名免费网站搭建：Hugo + PaperMod + Cloudflare Pages + Giscus 实战》一文来看。

---

## 十、常见问题与排查思路

### 问题 1：访问域名提示“网站未找到”或 522/525 错误？

排查顺序：

1. 在 Cloudflare → DNS 中确认对应记录是否存在（A/CNAME）；
2. 确认云朵是橙色还是灰色（是否走代理）；
3. 如果是指向自有服务器：
   - 先直接用 IP 访问，确认源站是否正常；
   - 再确认 SSL 模式是否与源站匹配（Full/Flexible）。

### 问题 2：更换 Nameserver 很久还没生效？

1. 在 `whois` 查询网站上查看 NS 是否已经变成 Cloudflare 的；
2. 确认注册商后台 NS 配置是否保存成功；
3. 一般最长不超过 24 小时，多数几分钟到半小时即可。

### 问题 3：自定义邮箱收不到邮件？

1. 在 Cloudflare Email Routing 中查看是否配置了正确的目标邮箱；
2. 检查 DNS 中是否存在 MX 记录指向 Cloudflare 的邮件服务器；
3. 检查垃圾邮件文件夹；
4. 尝试从不同邮箱地址发送测试邮件。

---

## 十一、总结：白嫖 Cloudflare 的最佳姿势

到这里，你已经完成了从“买了个域名”到“用好 Cloudflare 免费能力”的关键步骤：

- Cloudflare 成为你的域名权威 DNS，享受全局加速和防护；
- 配置了基础的 DNS 解析、SSL/TLS、HTTPS 强制跳转；
- 可选地配置了 Email Routing，拥有专业的域名邮箱；
- 为后续接入 Cloudflare Pages、Workers、KV、R2 等打好了基础。

接下来，你可以继续结合另一篇文章《域名免费网站搭建：Hugo + PaperMod + Cloudflare Pages + Giscus 实战》，  
把这个已经“白嫖到位”的 Cloudflare 域名，变成一个真正上线可用、可沉淀内容、可互动的个人技术网站。

---

## 相关阅读

- 如果你已经完成了 Cloudflare 的接入，现在可以继续看实战搭建文章：  
  [《域名免费网站搭建：Hugo + PaperMod + Cloudflare Pages + Giscus 实战》](/posts/域名免费网站搭建/)

