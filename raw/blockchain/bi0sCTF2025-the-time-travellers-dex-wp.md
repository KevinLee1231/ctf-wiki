# The Time Traveller DEX

## 题目简述

题目把一个 WETH/INR 恒定乘积池 `DEX` 与按时间加权价格兑换资产的 `Finance` 组合起来。`Setup.solve()` 要求调用者同时持有至少 `100000 ether` WETH、`230000 * 100000 ether` INR 和 `89835 ether` 原生 ETH；DEX 两侧储备不能低于初始值，且总 swap 次数最多 6 次。

真正的攻击面不是单一“闪电贷套利”，而是三处状态耦合：DEX 把用户声明的 `_amountIn` 用于限额，却把合约实际收到的 `_balanceIn` 用于计算输出；Finance 在 1 至 2 分钟的快照窗口内按 DEX 累计价格兑换；`withdraw` 又把本次检测到的全部超额 token 结算为 ETH。官方 `Solve.s.sol` 用六次分阶段广播，使价格快照、奖金、闪电贷和六次 swap 恰好同时满足最终约束。

## 解题过程

先看 DEX 的关键不一致。调用者必须先把 token 转进池子，`swap` 随后计算：

```solidity
uint256 balanceIn = currentBalance - reserveIn;
if (_amountIn > maxSwapToken) revert;
if (balanceIn < _amountIn) revert;

tokensOut = reserveOut
    - reserveIn * reserveOut / (reserveIn + balanceIn - fee);
```

限额检查只看 `_amountIn`，实际输出却使用 `balanceIn`；因此可以转入远多于限额的 token，再用封顶的 `_amountIn` 调用，输出仍按全部转入量计算。`Finance.getPrice()` 则用两次累计价格之差除以时间差，要求快照已满 1 分钟且未超过 2 分钟。攻击脚本必须跨多次交易等待这些窗口，而不能在一次调用里完成。

官方状态机按以下顺序推进。

第一阶段先 `finance.snapshot()`，领取无需资格的 `12500 ether` Bonus 1，并用这笔 ETH 部署 `FlashLoanReceiver`。接着闪借全部 `50000 * 230000 ether` INR。回调中依次：

1. 把借来的 INR 转入 DEX，执行第 1 次 swap 换出 WETH；
2. 把接收器的 `12500 ether` 原生 ETH 按当前快照价格 stake 成 INR；
3. 把全部 WETH 转回 DEX，执行第 2 次 swap 换回 INR；
4. 授权 Finance 收回本金，并把超出闪贷本金的 INR 转给主攻击合约。

第二阶段先 `dex.sync()` 固化新的储备与累计价格，然后：

```text
全部 INR -> Finance.withdraw(INR) -> ETH
50000 ETH -> Finance.stake(WETH) -> 50000 WETH
领取 10000 ETH Bonus 2
50000 WETH -> Finance.withdraw(WETH) -> 50000 ETH
33350 ETH -> Finance.stake(INR) -> INR
```

Bonus 2 要求调用者先持有 `50000 WETH`，所以领取顺序不能交换。

第三阶段创建下一段价格轨迹：先快照，把当前全部 INR 转进 DEX 并执行第 3 次 swap 换 WETH；再把 `26650 ether` 原生 ETH stake 成 INR；最后把全部 WETH 转回池中执行第 4 次 swap，结束后再次快照。

第四阶段 `dex.sync()` 后，将

$$
\frac{\text{INR balance}}{2}-9002\times230000\text{ ether}
$$

转给 Finance 并调用 `withdraw(INR, amount)`，按当前 INR/WETH 价格换回原生 ETH，同时留下后续所需的 INR。

第五阶段利用前述 `_amountIn`/`balanceIn` 差异。脚本先把全部 INR 转给 DEX，但传入的 `_amountIn` 只取：

$$
\min(\text{actual INR deposit},50000\times230000\text{ ether}).
$$

这样通过 max-swap 检查，输出计算却吃进全部实际余额，完成第 5 次 swap。随后把当前全部 ETH stake 成 INR，再把刚收到的 WETH 转回 DEX，执行第 6 次也是最后一次 swap。

第六阶段只做资产结算，不再 swap：

```solidity
IERC20(INR).transfer(address(finance), 189836 ether * 230000);
finance.withdraw(INR, 189836 ether * 230000);
finance.stake{value: 100000 ether}(WETH);
setup.setPlayer(address(this));
setup.solve();
```

这一步把一部分 INR 换成 ETH，再固定拿出 `100000 ether` ETH 铸成 WETH；剩余 INR 与 ETH 分别越过另外两条阈值。整个脚本的 swap 计数恰好是 $2+2+2=6$，而每轮回转及 `sync()` 又使 DEX 最终储备不低于初始值。

仓库脚本依赖分多次执行 `run()`：首次部署攻击合约并执行 `step1`，后续根据持久化的 `state` 每次推进一个步骤。这也是题名 “Time Traveller” 与快照窗口的实际含义。本次没有部署 Foundry 环境或等待真实时间窗口；上述金额、调用顺序、swap 计数和最终检查均由 `DEX.sol`、`Finance.sol`、`Setup.sol` 与官方 `Solve.s.sol` 逐项交叉核对。

## 方法总结

这道题应先画出三套状态：DEX 的实际余额/记录储备、Finance 的 `LatestBalances`/ETH 储备、以及累计价格快照。单看恒定乘积公式会漏掉 `_amountIn` 与 `balanceIn` 的语义错位，单看价格预言机又解释不了为什么能在 6 次交易内放大到目标资产。

复现时最重要的验收项是：每次快照是否落在 1 至 2 分钟窗口、Bonus 2 前是否已有 50000 WETH、六次 swap 是否按 2/2/2 分布、第五阶段是否“多转入但少申报”，以及最终三种资产与池储备是否同时达标。任何一步只看局部盈利而不检查这些全局约束，都会在 `solve()` 收口时失败。
