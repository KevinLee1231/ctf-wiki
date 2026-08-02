# front-door

## 题目简述

站点使用一个外形类似 JWT、实则自定义签名的 Cookie。任意用户都能创建临时账户取得“明文 header + 明文 payload + 64 字符签名”的已知消息样本；产品页公开签名算法，`robots.txt` 又公开 TODO 列表使用的 XOR 方案。恢复一个等价签名 key 后，把 payload 的 `admin` 从字符串 `false` 改为 `true`，即可访问隐藏的 `/business_secrets`。

## 解题过程

先创建普通账户并保存 `token` Cookie。产品页给出的签名算法为：

```python
def hash_char(a, b):
    return chr(pow(ord(a), ord(b), 26) + 65)

def sign(core, key):
    return "".join(
        hash_char(core[i % len(core)], key[i % len(key)])
        for i in range(64)
    )
```

签名的每个位置只依赖消息当前位置和周期 key 的一个字符，位置之间没有扩散。对捕获 token 的每个候选周期分别分组，逐字符检查小写 key 候选；第一个可行周期是 7。实际 key 不必唯一，因为模 26 幂运算会产生等价指数。在发布 token 上，各位置候选集合为：

```text
[['h', 't'], ['i', 'u'], ['l', 'x'], ['b', 'n', 'z'],
 ['e', 'q'], ['g', 's'], ['h', 't']]
```

取每组第一个字符得到等价 key `hilbegh`；它与源码中的真实 key `tuxbest` 对 token 使用的 Base64URL 字符集产生相同签名结果。下面的脚本从普通 token 自动找周期并生成管理员 token：

```python
import base64
import itertools
import string

normal_token = input("普通账户 token：").strip()
header, payload, signature = normal_token.split(".")
core = header + "." + payload

def hash_char(a, b):
    return chr(pow(ord(a), ord(b), 26) + 65)

def sign(message, key):
    return "".join(
        hash_char(message[i % len(message)], key[i % len(key)])
        for i in range(64)
    )

candidate_sets = None
for key_length in range(1, 17):
    groups = []
    for position in range(key_length):
        valid = [
            c for c in string.ascii_lowercase
            if all(
                hash_char(core[i % len(core)], c) == signature[i]
                for i in range(position, 64, key_length)
            )
        ]
        groups.append(valid)
    if all(groups):
        candidate_sets = groups
        break

assert candidate_sets is not None
equivalent_key = "".join(group[0] for group in candidate_sets)

admin_payload = b'{"username": "a", "password": "b", "admin": "true"}'
admin_payload_b64 = base64.urlsafe_b64encode(admin_payload).decode().rstrip("=")
admin_core = header + "." + admin_payload_b64
admin_token = admin_core + "." + sign(admin_core, equivalent_key)
print(equivalent_key)
print(admin_token)
```

`robots.txt` 中的 `ord(char) ^ 42` 提示可以解开 TODO 页的数字数组，其中关键一行是：

```text
Create "business_secrets" page -- made it but no button to access yet
```

将生成的 token 写入 Cookie 后请求 `/business_secrets`，服务端先验证签名，再从未加密 payload 中读取字符串 `admin == "true"`，最终返回：

```text
tjctf{buy_h1gh_s3l1_l0w}
```

## 方法总结

- 核心技巧：利用逐位置、周期性且无扩散的自定义签名，从一个已知消息—签名对恢复等价 key，再伪造管理员声明。
- 识别信号：自制 token 算法、可注册取得签名样本、payload 可读、权限只由客户端声明决定，以及隐藏路由线索使用弱 XOR 编码。
- 复用要点：认证 token 应采用成熟的 HMAC 或非对称签名方案并严格验证算法；权限还应与服务端账户状态绑定，不能只相信客户端 payload。
