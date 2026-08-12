# Hackergame2020 狗狗银行 WP

## 题目简述

银行提供储蓄卡、信用卡、转账和“过一天”等 API，净资产达到 2000 狗狗币时返回 flag。储蓄利息按整数四舍五入，而信用卡利息没有同样的归零漏洞。目标是利用大量小账户放大利息舍入误差。

## 解题过程

储蓄卡日利率是 $0.3\%$。每张卡存入 167 时，理论利息为：

$$
167\times 0.003=0.501,
$$

后端四舍五入成 1，实际日收益率约为 $1/167\approx0.599\%$。借款成本低于这个值，因此可以用信用卡借钱，再把资金拆成许多份 167 存入储蓄卡。

官方解题脚本反映了完整 API 顺序：

```python
post("reset")
post("create", type="credit")

for account in range(3, 203):
    post("create", type="debit")
    post("transfer", src=2, dst=account, amount=167)

for day in range(1, 37):
    post("eat", account=1)
    for account in range(3, 203):
        post("transfer", src=account, dst=2, amount=1)

post("eat", account=1)
print(get("user")["flag"])
```

每天把各储蓄卡新产生的 1 币利息取出，避免余额升高后舍入优势下降；一部分还信用卡，一部分支付每日消费。源码中的 `/user` 路由以精确大整数计算 `totalValue accounts`，只有结果至少为 2000 才返回 `flag{W0W.So.R1ch.Much.Smart.52f2d579}`，所以前端浮点显示误差不是解法。

## 方法总结

这是典型的业务逻辑与舍入攻击。逐账户舍入后再求和，与先求总利息再舍入并不等价；攻击者可以通过拆分账户把误差重复放大。金融类逻辑应统一最小货币单位、明确舍入时点，并限制可被批量制造的结算单元。
