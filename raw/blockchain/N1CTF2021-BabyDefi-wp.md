# N1CTF 2021 - BabyDefi

## 题目简述

题目部署了一组简化 DeFi 合约：`N1Token`、`FlagToken`、无手续费的恒定乘积兑换池 `SimpleSwapPair`、闪电贷合约和质押挖矿合约 `N1Farm`。目标账户既要保持质押余额大于 0，又要持有超过

$$
172800\times 10^{18}
$$

个 `FlagToken`，随后调用 `isSolved` 触发解题事件。

完整利用分两段：先围绕 `sellSomeForFlag()` 做三明治交易，从农场抽走大量 `N1Token`；再利用 `deposit()` 对旧 `PoolInfo` 快照记账的错误，让刚存入的大额本金也获得此前等待期间的奖励。

## 解题过程

### 利用农场的强制卖出制造三明治

`N1Farm.sellSomeForFlag()` 会把农场持有的全部 `N1Token` 卖进 `SimpleSwapPair`：

```solidity
function sellSomeForFlag() public {
    uint total = IERC20(tokenAccept).balanceOf(address(this));
    (uint112 reserveA, uint112 reserveB) = simpleSwapPair.getReserves();
    uint amountOut = getAmountOut(total, reserveA, reserveB);
    IERC20(tokenAccept).transfer(simpleSwapPair, total);
    simpleSwapPair.swap(0, amountOut, address(this), "");
}
```

该函数没有价格保护、最小输出量或可信调用者限制。攻击合约可在一笔闪电贷事务中完成：

1. 从 `FlashLoan` 借出其全部 `N1Token`。
2. 先把借款卖入兑换池，取走大量 `FlagToken`，扭曲储备比例。
3. 调用 `sellSomeForFlag()`，迫使农场在被操纵后的价格下卖出其全部 `N1Token`。
4. 把手里的 `FlagToken` 换回 `N1Token`，恢复价格并捕获农场刚注入池中的价值。
5. 归还闪电贷本金。

兑换池没有手续费，输出量为

$$
\operatorname{amountOut}=\left\lfloor
\frac{\operatorname{amountIn}\cdot R_{out}}{R_{in}+\operatorname{amountIn}}
\right\rfloor.
$$

官方 `Exp.sol` 的 `launch()` 与 `flashLoanCall()` 实现了上述原子操作。执行后攻击合约约可获得 $514\times 10^{18}$ 个 `N1Token`，为第二阶段提供本金。

### 利用旧快照放大奖励

`deposit()` 在更新池之前，把结构体复制到了内存：

```solidity
PoolInfo memory poolInfo = poolInfos[token]; // 旧快照
updatePool(token);                          // 更新 storage
...
user.rewardDebt = user.amount
    .mul(poolInfo.accRewardsPerToken)       // 仍使用旧值
    .div(1e18);
```

正常逻辑应使用更新后的 `poolInfos[token].accRewardsPerToken` 记录 `rewardDebt`。当前实现却让新余额乘以旧累计奖励值，造成债务少记。

利用顺序如下：

1. 先批准并存入 1 wei 的 `N1Token`，初始化用户和池的时间戳。
2. 等待约 6 分钟，让实际 `accRewardsPerToken` 随时间增长。
3. 把第一阶段取得的其余 `N1Token` 全部存入。该调用虽然执行了 `updatePool()`，却仍按更新前的快照计算整个大额余额的 `rewardDebt`。
4. 调用 `claimRewards()`。此函数使用 storage 中的最新累计奖励值，于是把“新存入的大额本金 × 等待期间累计奖励”也计为待领取奖励，并调用 `FlagToken.mint()`。

关键差额可以写成

$$
\operatorname{pending}
=A_{large}\cdot acc_{new}-A_{large}\cdot acc_{old},
$$

而正常情况下新增本金不应分享 $acc_{old}\rightarrow acc_{new}$ 这段历史奖励。奖励超过目标后调用 `isSolved(attacker)` 即可完成题目。

## 方法总结

本题把两类常见 DeFi 缺陷串在一起：公开的无滑点保护大额交易可被闪电贷三明治，质押合约又因“先复制、后更新”而混用了旧状态。审计这类合约时，应同时检查价格敏感函数是否有最小输出与权限约束，以及 `accRewardPerShare/rewardDebt` 是否在同一个状态快照上计算。
