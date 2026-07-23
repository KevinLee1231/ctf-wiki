# PP Farming 2

## 题目简述

第二版为 `withdrawPP()` 增加了 `noReentrancy`，并把提款处理委托给辅助合约。ATM 的回退函数还允许把其他调用 `delegatecall` 到该辅助合约。目标仍是清空 ATM。

## 解题过程

直接重入已经会被 `locked` 阻止，因此需要检查代理调用的存储布局：

```text
slot 0: ATM scores mapping       / Helper id_number
slot 1: ATM helper address       / Helper atm
slot 2: ATM locked               / Helper helping
```

`delegatecall` 执行的是 Helper 代码，但写入的是 ATM 自己的存储。Helper 的 `stopHelping()` 会把它认为的 `helping` 设为 `false`；经 ATM fallback 委托执行后，实际被清除的是 ATM 的 `locked`。

提款本身也通过委托调用完成：

```solidity
(bool success, ) = performancePointHelper.delegatecall(
    abi.encodeWithSignature(
        "processWithdrawal(address,uint256)",
        msg.sender,
        score
    )
);
```

因此 Helper 中的 `recipient.call{value: amount}` 实际从 ATM 的余额付款，并把控制权交给攻击合约。ATM fallback 只禁止直接调用 `processWithdrawal(address,uint256)` 的 selector，其余 Helper 公共函数仍可任意委托执行。

攻击流程如下：

1. 攻击合约捐赠，获得非零 `score`。
2. 调用 `withdrawPP()`，修饰器把 ATM 的 `locked` 设为 `true`。
3. ATM 向攻击合约付款，进入攻击合约的 `receive()`。
4. 对 ATM 调用接口中的 `stopHelping()`；ATM 本身没有该函数，因此进入 fallback。
5. `delegatecall` 将 ATM slot 2 写成 `false`。
6. 再次调用 `withdrawPP()`，重入锁已被清除，可以继续提款。

官方攻击合约中的关键调用是：

```solidity
atm.stopHelping();
atm.withdrawPP();
```

与第一题相同，捐赠额应选择能整除初始余额的数值，例如 1 ETH；否则最后一次重入可能因 ATM 剩余余额小于 score 而使整笔交易回滚。攻击完成后用 `address(atm).balance == 0` 或 `isSolved()` 验证确已清空。

最终得到：

```text
SEKAI{pr0xie5_4r3_h4rD_2_3t4k3}
```

## 方法总结

代理合约的安全性取决于代码与存储布局的共同约束。`delegatecall` 暴露任意辅助函数时，辅助合约中看似无害的状态变量写入可能覆盖代理的锁、管理员或实现地址。重入锁本身没有写错，问题是攻击者能从另一条委托路径直接改掉它。
