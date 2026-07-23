# skyblock

## 题目简述

题目运行了一套 Minecraft Skyblock 经济系统：NPC 商店负责固定价买卖，`/tradebook` 则由自动做市机器人补充流动性。目标是获得 $10^{12}$ coins，并从 Flag Merchant 购买不可交易的 `ctf_flag`。

题目的决定性问题不是物品复制，而是同一物品在商店、合成与做市模块中使用了互相矛盾的价格。官方解法只提示“从商人购买 fishing rod，再卖到 tradebook”，源码还能还原出后续的一次性放大链。

## 解题过程

### 确认做市机器人的收购条件

`items/fishing_rod.yml` 将鱼竿的 NPC 购买价设为 100 coins，而 `market-making.yml` 又为同一个 `fishing_rod` 指定了 750000 的参考价：

```yaml
# items/fishing_rod.yml
buy-price: 100
sell-price: 10

# market-making.yml
fishing_rod:
  reference-price: 750000
  target-active-units: 8
```

`MarketMakingBotService` 会先根据全服货币量修正参考价，再混入近期玩家成交价或最低挂单价，最后用公平价的 85%～95% 计算机器人买价。由于鱼竿在做市配置中有独立 `reference-price`，`demandPriced()` 为真，代码不会再用 NPC 的 100 coins 价格限制机器人买价。

源码同时给动态价格设置了下界：

- 通胀价最低为参考价的 85%；
- 玩家市场价最低按参考价的 50% 计；
- 玩家市场价的混合权重最高为 35%；
- 机器人买价最低为混合公平价的 85%。

因此即使把所有系数都取成对玩家最不利的值，机器人买价仍不低于参考价的

$$
\left(0.65\times0.85+0.35\times0.5\right)\times0.85
=0.618375.
$$

鱼竿对应的保守阈值为 $750000\times0.618375\approx463781$ coins。玩家只要把鱼竿按不高于该阈值的单价挂单，机器人就会把它识别为低价订单并买入。机器人启动后 15 秒进行首轮做市，之后每 90 秒运行一次；成交款还需在 `/tradebook` 中点击 `Claim Coins` 领取，并扣除 5% 交易税。

### 用鱼竿取得启动资金

1. 先通过正常采集获得至少 100 coins。
2. 在 Fishing Merchant 处花 100 coins 购买一根 Fishing Rod。
3. 在 `/tradebook` 中将鱼竿以 450000 coins 挂单。
4. 等待下一次做市周期。挂单成交后点击 `Claim Coins`，税后得到 427500 coins。
5. 按需重复该过程，直到可以买到后续合成所需的材料。

450000 低于源码可推导出的保守收购阈值，因此不依赖某一轮随机价差。若现场盘口允许，也可以提高挂单价来减少循环次数。

### 合成并高价卖出 Speedster Boots

只靠鱼竿重复累积到 $10^{12}$ 很慢。真正的放大点是 `Speedster Boots`：

```yaml
# items/speedster-set/speedster_boots.yml
sell-price: 102400
crafting:
  shape:
    - "A A"
    - "A A"
    - "   "
  ingredients:
    A:
      item: enchanted_sugar_cane
      amount: 1

# market-making.yml
speedster_boots:
  reference-price: 1500000000000
  target-active-units: 1
```

做市系统中 `Enchanted Sugar Cane` 的参考价回退到物品的 102400 coins。每双靴子需要 4 个，先通过鱼竿套利取得足够资金，再从机器人挂单中买入至少 8 个并合成两双。材料价格也会随全服货币量变化，因此应以当前盘口为准，不足时继续卖鱼竿。合成出的 `Speedster Boots` 被认定为玩家可生产物品，但其做市参考价被错误写成 $1.5\times10^{12}$。

沿用前面的最保守系数，靴子的机器人收购阈值仍不低于

$$
1.5\times10^{12}\times0.618375
=9.275625\times10^{11}.
$$

将两双靴子分别以 $6\times10^{11}$ coins 挂单，两笔都明显低于该保守阈值。成交后扣除 5% 税，合计到账为

$$
2\times6\times10^{11}\times0.95
=1.14\times10^{12},
$$

已经足够支付 Flag 的 $10^{12}$ coins 价格。完整操作为：

1. 用鱼竿套利获得启动资金。
2. 在 `/tradebook` 买入 8 个 `Enchanted Sugar Cane`。
3. 按靴子形状合成两双 `Speedster Boots`。
4. 将两双靴子分别以 600000000000 coins 挂单，等待机器人买入。
5. 领取税后成交款，再到 Flag Merchant 购买 `ctf_flag`。

购买时服务端显示：

```text
SEKAI{sekai-craft-is-back-in-business-:sunglasses:}
```

## 方法总结

这是一条两级跨模块套利链：先利用鱼竿的 100/750000 价差取得启动资金，再利用 `Speedster Boots` 的低成本合成配方和 $1.5\times10^{12}$ 做市参考价越过最终门槛。分析此类经济题时，应把 NPC 买卖价、合成成本、做市参考价、动态系数、市场反馈权重和税率放在同一张表中，并检查是否存在从低价来源走到高价收购端的闭环。

修复时应让商店与做市系统共享可信价格源，并在做市机器人收购玩家可生产物品前，用可获得的最低生产成本约束买价。仅限制盘口波动或收取少量交易税，无法消除数量级错误造成的无风险套利。
