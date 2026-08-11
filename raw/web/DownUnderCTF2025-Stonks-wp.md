# stonks

## 题目简述

这是一个 Flask 虚拟货币兑换站。用户余额保存在服务器进程内，但当前币种来自客户端持有的 Flask session cookie。session 虽有签名、不能任意伪造，却是无服务器状态的快照；服务端没有校验 cookie 中的旧币种是否与账号当前币种一致，因此可以重放旧 cookie，让同一笔余额反复套用高倍率汇率。

## 解题过程

兑换公式为：

$$
B_{new}=\frac{B_{old}}{R_{old}}\times R_{new},
$$

其中 $R_{old}$ 直接取自 `session["currency"]`，而更新后的余额写回服务器端 `user_balances`：

```python
old_currency = session["currency"]
new_currency = request.form["currency"]

session["currency"] = new_currency
user_balances[u] = (
    user_balances[u] / CURRENCY_CONVERSIONS[old_currency]
) * CURRENCY_CONVERSIONS[new_currency]
```

利用步骤如下：

1. 注册并登录，先把 AUD 余额兑换成 GBP。
2. 保存此时服务器返回的、币种字段为 `GBP` 的合法 session cookie。
3. 使用这枚旧 cookie 请求 `POST /change-currency`，目标币种选择 `IDR`。
4. 忽略响应中更新为 IDR 的新 cookie，继续重放保存的 GBP cookie，重复同一请求。

每次请求都会让服务器端余额乘以：

$$
\frac{10597.38}{0.48}\approx 22077.875.
$$

cookie 仍声称旧币种是 GBP，而余额已经被上一次请求放大，所以倍率可以连续叠加。余额超过 $10^{12}$ 后，携带保存的 cookie访问 `/are-you-rich`；该路由再次依据 session 币种折算 AUD，满足阈值并返回 flag。

## 方法总结

- 核心技巧：重放旧的合法客户端状态，使其与持续变化的服务器端余额发生版本错配。
- 识别信号：Flask session 保存影响金额计算的币种，而余额和账号当前币种另存服务端；兑换时只信任 cookie，没有做一致性校验。
- 复用要点：签名只能保证 cookie 未被篡改，不能阻止旧状态重放。金额计算应以服务端权威状态为准，或给状态加入单调版本号并拒绝过期快照。
