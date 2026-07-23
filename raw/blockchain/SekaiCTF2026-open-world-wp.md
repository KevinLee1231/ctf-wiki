# Open World

## 题目简述

题目在 TON 上实现了一个 Jetton 经济系统。每个实例都可以领取一次 50 枚 Jetton，商店以固定价格买卖代币；向挑战合约发送 100 枚 Jetton 并携带 `Solve` 消息即可通过。

关键常量为：

- 领取奖励：50 Jetton；
- 兑换目标：100 Jetton；
- 买入价格：每枚 2 TON；
- 买入消息还要额外携带 0.12 TON 的铸币费用；
- 卖出时按固定价格退还 TON。

合约中的约束可精简为：

```tolk
const TOKEN_PRICE: coins = ton("2")
const FLAG_PRICE: coins = 100
const BUY_MINT_TON_AMOUNT: coins = ton("0.12")

assert (
    in.valueCoins > msg.amount * TOKEN_PRICE + BUY_MINT_TON_AMOUNT
) throw Errors.INSUFFICIENT_FUNDS
```

单个新实例的初始资金不足以直接买齐缺少的 50 枚 Jetton，但不同实例之间的 TON 可以自由转账。

## 解题过程

先阅读 `Challenge.tolk`、Jetton 合约和官方 `solve.ts`。问题不在签名或整数溢出，而在实例之间只隔离了挑战状态，没有隔离链上的基础资产。

第一份实例作为资金提供者：

1. 领取 50 枚免费 Jetton。
2. 将 50 枚 Jetton 卖回商店。
3. `Sell` 分支按 `jettonAmount * TOKEN_PRICE` 构造付款，并使用 `SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE` 发送；50 枚 Jetton 因而换回约 100 TON。
4. 把这些 TON 转给第二份目标实例的钱包。

第二份实例执行：

1. 领取本实例的 50 枚免费 Jetton。
2. 使用外部转入的 100.2 TON 买入另外 50 枚 Jetton；该数值同时覆盖 100 TON 价格、0.12 TON 铸币费用和消息执行余量，并满足源码中的严格大于判断。
3. 钱包余额达到 100 Jetton 后，将其全部发送给挑战合约，并在转账负载中编码 `Solve`。

逻辑可以概括为：

```text
donor instance:
    claim 50 jettons
    sell 50 jettons
    transfer received TON to target wallet

target instance:
    claim 50 jettons
    buy 50 jettons
    send 100 jettons with Solve payload
```

官方脚本在每一步都等待链上状态变化：捐赠实例卖出后余额必须足够转出 100.2 TON，目标实例的 Jetton 余额必须达到 100，最后轮询 `getIsSolved()`。因此成功不是根据交易是否被接受推测，而是由挑战合约的持久状态确认。

成功后得到：

```text
SEKAI{3Xp1or1ng-An-0pen-W0rld-15-FUN}
```

## 方法总结

漏洞属于跨实例经济隔离缺失：实例化平台隔离了题目合约，却没有阻止玩家把同一条公链上的 TON 从一个实例转入另一个实例。分析链上经济题时，不能只检查单份合约内的守恒关系，还要把外部账户、其他实例和基础资产转账纳入威胁模型。
