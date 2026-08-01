# GlacierCTF 2024 Frozymarket

## 题目简述

题目是一个支持创建多个二元预测市场的 Solidity 合约。`Setup` 在市场 0 的正反两边各下注 10 ETH，目标是清空整个合约的 20 ETH 初始资金。

每个市场分别记录下注额，但兑奖函数没有使用该市场自己的资金池，而是按获胜者在局部市场中的占比瓜分合约的全局余额。攻击者可以自建一个只有自己下注的市场，以 100% 局部份额领取其他市场的全部资金。

## 解题过程

### 1. 找到局部记账与全局付款的错位

市场结构分别维护 `totalBetsA`、`totalBetsB`，获胜者占比也只根据当前市场计算：

```solidity
bpsOfPot = userBet * BPS / winningSideTotal;
```

但实际付款使用的是：

```solidity
payout = address(this).balance * bpsOfPot / BPS;
```

因此市场 ID 只影响比例，不限制可领取的资产范围。只要在任一市场获得 100% 的获胜份额，`payout` 就等于整个合约余额。

### 2. 创建可立即结算的独占市场

`createMarket()` 对任何地址开放，创建者还是该市场的 owner。令 `resolvesBy = 0`，当前时间天然满足结算条件，然后：

1. 创建市场 1；
2. 仅在 A/true 一侧下注 1 ETH；
3. 由市场 owner 将结果结算为 true；
4. 调用 `claim()`。

这一市场的获胜侧总额和攻击者下注额都是 1 ETH，所以：

$$
\mathrm{bpsOfPot}=\frac{1\ \mathrm{ETH}}{1\ \mathrm{ETH}}\times10000=10000.
$$

此时合约余额为原有 20 ETH 加攻击者的 1 ETH，兑奖函数一次转出全部 21 ETH，目标合约余额归零。仓库中的 flag 为：

```text
gctf{m0m_I_finally_m4d3_m0ney_g4mbl1ng_0n_th3_bl0ckch41n}
```

## 方法总结

本题的决定性缺陷是“局部份额、全局资金”混用。多市场系统必须为每个市场维护独立资金池，兑奖金额只能来自对应市场，并在结算后保持总付款不超过该池余额。审计类似合约时，应沿着市场 ID 检查从下注、统计、结算到转账的每一步；任何一步退回 `address(this).balance` 都可能跨池窃取资金。
