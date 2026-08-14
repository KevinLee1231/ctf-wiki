# Vastavikamaina Token

## 题目简述

`Setup` 创建三组 VSTETH/LamboToken Uniswap V2 池，并向 `Balancer` 存入约 `32560.2 WETH`。最终条件很简单：指定 `player` 的原生 ETH 余额至少为 `141.3 ether`。旧版可以直接把 `player` 设为 WETH 合约地址；当前源码已经禁止这一地址，并禁止在闪电贷尚未结束时调用 `setPlayer`。

预期攻击利用 `Factory.addVasthavikamainaLiquidity`：它可以向现有池子借入并直接铸造最多 `300 VSTETH`，同时把债务记在 pair 上。攻击者借出 Balancer 的全部 WETH，用它操纵三条池子，把无真实 ETH 抵押的贷款 VSTETH 兑换成 `WhiteListed` 合约中已有的 ETH，再归还闪电贷，把差额逐次转给 player。

## 解题过程

`VasthavikamainaToken.takeLoan(pair, amount)` 同时执行两件事：向 pair 铸造 VSTETH，并增加 `_debt[pair]`。代币的 `_update` 又要求转出后余额不能低于债务：

```solidity
if (from != address(0) && balanceOf(from) < value + _debt[from]) {
    revert VasthavikamainaToken__DebtOverflow(...);
}
```

所以攻击不能让 pair 转走那 300 VSTETH 债务本身，但可以抽走 `reserveVSTETH - debt` 的全部净余额。

官方 `FlashLoanReceiver` 的一次回调针对一条池执行：

1. 闪借 Balancer 当前全部 WETH，并 `withdraw` 成相同数量的 ETH；
2. 调用 `WhiteListed.buyQuote`，把这些 ETH 包装成 VSTETH 放进目标池，换出 LamboToken；
3. 调用 `Factory.addVasthavikamainaLiquidity(VSTETH, lamboToken, 300 ether, lamboOut)`；Factory 向 pair 贷款 300 VSTETH，按现有储备比例从攻击者取走 LamboToken，并把 LP 份额铸给零地址；
4. 读取 pair 储备和 `VSTETH.getLoanDebt(pair)`，令

$$
V_{net}=R_{VSTETH}-D_{pair};
$$

5. 用 Uniswap V2 反向报价计算为了换出 $V_{net}$ 所需的 LamboToken：

$$
L_{in}=\left\lfloor
\frac{R_L\cdot V_{net}\cdot1000}
{(R_{VSTETH}-V_{net})\cdot997}
\right\rfloor;
$$

6. 调用 `WhiteListed.sellQuote(lamboToken, L_in, 0)`。它从 pair 换出 VSTETH，再调用 `cashOut` 烧毁 VSTETH并支付真实 ETH；
7. 从所得 ETH 中取出原闪贷数量，重新包装为 WETH 并转回 Balancer，剩余 ETH 即为本轮利润。

贷款 VSTETH 没有对应的新 ETH 进入 `VasthavikamainaToken`，却改变了池储备和可兑换路径；`sellQuote` 最终从合约已有的 ETH 余额付款，因此每轮都把存量 backing 转成攻击者利润。300 VSTETH 的每区块贷款上限使脚本把三条池拆成三次独立广播。`FlashLoanReceiver.state` 持久化选择 `lamboToken1/2/3`，每次回调结束都把当前利润发送给 `player`；第三次之后 player 的余额超过 `141.3 ether`。

当前仓库还有一处必须如实记录的版本偏差：官方 `Solve.s.sol` 在第三次闪贷回调内部调用 `setup.setPlayer(player)`，但当前 `Setup.setPlayer` 明确在 `balancer.flashLoanTaken()==true` 时回滚。要与当前源码一致，应让回调先正常返回、等待 Balancer 把 `flashLoanTaken` 重置为 `false`，再从外层交易调用：

```solidity
balancer.flashloan(flashLoanReceiver, tokens, amounts, data);
setup.setPlayer(player);  // 必须位于 flashloan 返回之后
require(setup.isSolved(), "challenge unsolved");
```

`setPlayer` 没有“一次性”限制，因此可以每轮闪贷返回后都设置同一 player，最终第三轮余额达标即可。该调整不改变资金攻击，只修正官方脚本与当前防护版本的调用时机。

本次没有部署 Foundry 或运行三轮链上交易；闪贷资金来源由 `Deploy.s.sol` 核对，三池状态机、贷款/债务约束、报价公式和上述版本偏差均由当前源码与官方 exploit 交叉确认。

## 方法总结

本题的核心不是 Balancer 本身少做了还款检查；闪贷本金确实被完整归还。问题在于 Factory 能把带债务的 VSTETH 注入 AMM，`WhiteListed` 又把从 AMM 得到的 VSTETH 按 1:1 兑换为真实 ETH，导致无抵押贷款可以抽走旧有 backing。

分析这类金融组合题时，应分别跟踪 token 余额、pair 储备、债务映射与真实 ETH backing，不能把“池里多了 300 token”误当作“系统资产多了 300 ETH”。同时要沿当前源树检查 exploit 的调用时机；旧脚本在新 guard 下会回滚时，应明确给出最小兼容调整，而不能宣称原脚本已直接通过。
