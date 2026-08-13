# Voting Vault

## 题目简述

用户锁定 GREY 后获得 $1.3$ 倍投票权，并可把票委托给其他地址。通关需要通过提案转走金库中的 10000 GREY，而正常玩家远达不到 1000000 GREY 的票数门槛。漏洞来自两种计算粒度不同：每次锁仓分别向下取整，但委托时先汇总金额再计算；差额在 unchecked 的减票逻辑中造成下溢，并被截断存入 `uint224` checkpoint。

## 解题过程

对 1 wei 调用 `_calculateVotes()`：

$$
\left\lfloor\frac{1\times1.3\cdot10^{18}}{10^{18}}\right\rfloor=1.
$$

连续锁定 1 wei 十次，每次独立取整，当前地址累计只有 10 票。委托函数却先从累计存款算出总额 10，再统一计算：

$$
\left\lfloor\frac{10\times1.3\cdot10^{18}}{10^{18}}\right\rfloor=13.
$$

于是委托时会从原地址扣除 13 票：

```solidity
for (uint256 i = 0; i < 10; i++) vault.lock(1);
vault.delegate(address(1));
```

`_subtractVotingPower()` 位于 `unchecked` 块，$10-13$ 在 256 位整数中下溢；`History.push()` 再把结果转换成 `uint224`，最终 checkpoint 为 $2^{224}-3$。这已经远大于提案门槛。

第一笔交易中创建把金库全部 GREY 转给玩家的提案。`Treasury.vote()` 读取 `block.number - 1` 的投票权快照，所以投票必须放在下一块：

```solidity
// 第一个区块
proposalId = treasury.propose(address(grey), 10_000e18, player);

// 下一个区块
treasury.vote(proposalId);
treasury.execute(proposalId);
```

执行提案后得到：

```text
grey{rounding_is_dangerous_752aa6bb8b6a9f61}
```

## 方法总结

分项取整之和不一定等于汇总后取整，尤其在乘固定比例时会产生可累积误差。任何“先按旧值减去重新计算结果”的逻辑都应保证两边采用完全相同的计量粒度，并避免 unchecked 下溢。投票系统还需要关注 checkpoint 的位宽截断和快照区块，否则一个很小的舍入差就能放大成几乎最大的投票权。
