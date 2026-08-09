# Chronostasis

## 题目简述

题目模拟一个超额抵押稳定币协议，包含深流动性的 TKA/TKB 池、薄流动性的 TKB/TKC 池、8 个观测值的 300 秒滑动 TWAP，以及一个 EIP-7540 风格的异步 LP 金库。`requestRedeem` 按请求时价格记录份额快照，`claimRedeem` 却按领取时的 LP 现价计算输出。

胜利条件是让金库持有的 A/B LP 低于初始值。漏洞本质是“分子使用旧的可操纵高价，分母使用恢复后的正常价”，再叠加薄池、短窗口和任何人可更新预言机，攻击者能够赎回远超公平份额的 LP。

## 解题过程

### 1. 明确价格与结算公式

组合预言机先从 B/C 池计算 TKB 的美元价格，再经 A/B 池换算 TKA 价格，最后估算 LP 单价。异步赎回近似为：

$$
\mathrm{lpOut}=\frac{\mathrm{shares}\times\mathrm{snapshotPricePerShare}}
{\mathrm{currentLPPrice}}.
$$

若请求阶段的快照被抬高，而领取阶段当前价恢复正常，则 $\mathrm{lpOut}$ 会被同比放大。

### 2. 准备份额并抬高 TKB 价格

先补足预言机所需的初始观测值，再用手中的 TKA、TKB 向深池添加流动性，把得到的 A/B LP 存入金库换取 shares。

随后从薄的 B/C 池闪电借出约 95% 的 TKB：

```solidity
(uint112 r0, uint112 r1,) = pairBC.getReserves();
uint256 amount = uint256(r1) * 95 / 100;
pairBC.swap(0, amount, address(this), bytes("flash"));
```

回调中用 TKC 支付 Uniswap V2 所需输入。交换结束后池中 TKB 储备只剩很小一部分，TKB/TKC 现货价格显著升高。

### 3. 用异常观测覆盖 TWAP

仅操纵一次现货价不够，必须让异常状态覆盖 300 秒窗口。推进区块时间并连续调用 8 轮 `oracle.update(pairBC)` 和 `oracle.update(pairAB)`，每轮保持合理间隔，使环形缓冲区的有效观测全部落在异常价格阶段。

此时调用：

```solidity
uint256 requestId = vault.requestRedeem(
    shares,
    address(this),
    address(this)
);
```

金库把被抬高的 `snapshotPricePerShare` 固定在该请求中。

### 4. 恢复价格后领取

把持有的 TKB 反向换回 TKC，使 B/C 储备比例回到接近初始状态。再次推进时间并写入 8 轮正常观测，令 TWAP 恢复。最后调用 `claimRedeem(requestId)`；结算公式中的分子仍是异常快照，分母却已经是正常 LP 价格，因此从金库取出的 LP 超过存入份额应得数量。

官方 `exp/Exploit.sol` 实现攻击合约，`exp/exp.py` 负责部署、推进时间和发送交易。复现时应在请求前后分别读取 `pricePerShare()`，确认高价快照与正常结算价确实形成差值。

## 方法总结

本题不是普通的单交易闪电贷，而是跨时间的预言机状态机攻击。审计异步金库时，要核对请求与领取是否使用同一估值基准，并检查 TWAP 的流动性、最小观测间隔、窗口覆盖成本和更新权限。这里 8 个观测值不是装饰参数：只有完整覆盖环形缓冲区，异常价格才会稳定进入快照；恢复阶段也必须做同样工作，才能放大最终赎回量。
