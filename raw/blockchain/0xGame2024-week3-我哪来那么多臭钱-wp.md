# 我哪来那么多臭钱？？

## 题目简述

合约使用 Solidity 0.7.0。调用者可通过 `Get()` 把内部余额设为 50，但 `check()` 要求余额恰好为 114514。`Transfer()` 同时要求调用者地址最低字节为 `0xef`。解题需要生成满足后缀的 EOA，并利用 Solidity 0.8 以前默认不检查的 `uint256` 下溢，把 50 一次变成 114514。

## 解题过程

### 1. 解出转账金额

核心代码为：

```solidity
require(balance[msg.sender] - amount > 0, "Mamba out!");
require(uint160(msg.sender) % 256 == 239, "who am i?");
balance[msg.sender] -= amount;
balance[to] += amount;
```

在 Solidity 0.7.0 中，`uint256` 算术默认按模 $2^{256}$ 回绕。调用 `Get()` 后余额为 50，希望减法结果为 114514，因此：

$$
50-amount\equiv114514\pmod{2^{256}}
$$

解得：

$$
amount=2^{256}-114514+50=2^{256}-114464
$$

对应十进制和十六进制为：

```text
115792089237316195423570985008687907853269984665640564039457584007913129525472
0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe40e0
```

此时 `balance[msg.sender] - amount` 在 `require` 和实际赋值中都会回绕为 114514，因此检查可以通过。

### 2. 生成末字节为 `ef` 的地址

`uint160(msg.sender) % 256 == 239` 只检查地址的最低 8 位；$239=0xef$，所以 EOA 的十六进制地址必须以 `ef` 结尾。平均枚举约 256 个私钥即可找到，不需要依赖第三方网页：

```python
import secrets
from eth_account import Account

while True:
    private_key = "0x" + secrets.token_hex(32)
    account = Account.from_key(private_key)
    if int(account.address, 16) % 256 == 0xEF:
        print(f"address     = {account.address}")
        print(f"private_key = {private_key}")
        break
```

这里只应使用脚本新生成的比赛专用私钥，不能把真实钱包私钥交给网页或写进 WP。

### 3. 完成合约调用

将生成的私钥导入比赛链的钱包，并确保以下交易全部由该 `...ef` 地址发出：

1. 调用 `Get()`，确认 `balance(sender) == 50`；
2. 调用 `Transfer(to, amount)`，其中 `to` 必须是不同于 sender 的任意地址，`amount` 使用上面的巨大整数；
3. 查询 `balance(sender)`，应为 `114514`；
4. 调用 `check()`；
5. 调用 `isSolved()`，返回 `true`。

仓库的赛题记录给出的 flag 为：

```text
0xGame{17803c13-ad8d-4e00-bf7b-f3f0ad057b3d}
```

## 方法总结

本题把两个独立约束组合在一起：低版本 Solidity 的无符号整数下溢，以及调用者地址低字节约束。前者通过模方程直接求出金额，后者只需本地枚举虚荣地址。`to` 不能填写发送者自身，否则后续 `balance[to] += amount` 会再次修改同一余额，破坏 114514 的目标值。
