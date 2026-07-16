# another signin

## 题目简述

题目要求与已部署的 `Greeter` 合约交互：先把 greeting 设置为固定字符串 `Love0xGame`，再触发 `isSolved()`。利用合约把两次调用封装在同一个调用者身份中，部署时只需传入题目实例地址。

## 解题过程

根据题目给出的 ABI，只声明实际使用的两个外部函数。攻击合约保存挑战实例，并在 `hack()` 中按顺序调用：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGreeter {
    function setGreeting(string calldata greeting) external;
    function isSolved() external;
}

contract Attack {
    IGreeter public immutable target;

    constructor(address targetAddress) {
        target = IGreeter(targetAddress);
    }

    function hack() external {
        target.setGreeting("Love0xGame");
        target.isSolved();
    }
}
```

在 Remix 或本地开发框架中编译并部署 `Attack`，构造参数填写题目生成的目标合约地址；随后调用无参数的 `hack()`，等待交易确认，再回到平台检查 solved 状态。

这里的类型转换 `IGreeter(targetAddress)` 不会部署新合约，只是让编译器按已知 ABI 对该地址编码外部调用。两次调用都由 `Attack` 发出，因此目标合约看到的 `msg.sender` 保持一致。

## 方法总结

基础合约交互题应先从 ABI 提取函数签名、参数和调用顺序，再决定使用 EOA 直接调用还是中间合约。正文已包含完整接口、固定字符串和部署参数；题目源码未随该拆分材料提供，因此不额外臆测 `isSolved()` 内部未公开的判断细节。
