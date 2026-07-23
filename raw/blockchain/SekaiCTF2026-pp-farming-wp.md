# PP Farming

## 题目简述

`PerformancePointATM` 接受 ETH 捐赠，并把捐赠金额记入调用者的 `score`。调用 `withdrawPP()` 时，合约先按 `score` 向调用者转账，之后才把 `score` 清零。挑战合约初始持有 10 ETH，目标是清空其余额。

## 解题过程

危险顺序如下：

```solidity
uint256 score = scores[msg.sender];
require(score > 0, "Nothing to withdraw");
(bool result, ) = msg.sender.call{value: score}("");
require(result, "Transfer failed");
scores[msg.sender] = 0;
```

外部调用发生在状态更新之前。攻击合约先通过 `donatePP(address(this))` 给自己记入一笔分数，再调用 `withdrawPP()`。收到 ETH 时触发 `receive()`，此时 `scores[address(this)]` 尚未清零，于是可以再次进入 `withdrawPP()`。

官方 `Attacker.sol` 的回调逻辑为：

```solidity
receive() external payable {
    if (address(atm).balance >= msg.value) {
        atm.withdrawPP();
    }
}
```

每层回调收到的 `msg.value` 就是最初登记的 score。若选择 1 ETH，则 ATM 原有的 10 ETH 加上攻击者刚捐入的 1 ETH，恰好能完成 11 次提款；一般而言，捐赠额应整除初始 10 ETH，避免最后剩余金额小于 score 而无法继续提款。最外层调用返回后，各层才依次执行 `scores[msg.sender] = 0`，已经无法挽回转出的资金。

官方攻击合约最后同时检查目标余额为 0，挑战合约的 `isSolved()` 也只在 `address(this).balance == 0` 时返回真。

最终得到：

```text
SEKAI{3Z_re3ntr4ncy_atTack5}
```

## 方法总结

这是标准的 Checks-Effects-Interactions 违例。所有依赖余额、积分或份额的提款逻辑，都应先更新内部状态，再进行外部调用，并辅以可重入锁。仅检查 `call` 的返回值并不能阻止重入。
