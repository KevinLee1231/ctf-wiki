# Race

## 题目简述

Setup 创建一场一年后开始的比赛，奖池为 $1000$ GREY，报名费为 $100$ GREY；玩家可领取 $500$ GREY，目标是最终持有至少 $1500$ GREY。

`claimPayout` 使用 `delete races[raceId].players` 清理 OpenZeppelin 风格的 `EnumerableSet`，但删除包含动态 mapping 的结构不会清除 mapping 槽，导致数组与索引映射失去同步。

## 解题过程

先领取 $500$ GREY，用全部资金创建第二场比赛：奖池 $500$ GREY、持续时间 1 秒、报名费为零。关键不变量是 `EnumerableSet` 用数组保存元素，同时用 mapping 保存“数组下标 + 1”；直接 `delete` 只清空数组长度，历史 mapping 值仍然存在。

在比赛尚未开始的同一笔交易中报名：

```solidity
setup.claim();
setup.grey().approve(address(setup.race()), type(uint256).max);
uint256 raceId = setup.race().createRace(500e18, 1, 0);
setup.race().enterRace(raceId);
```

下一块时间戳超过 `startTime` 后，调用一次 `claimPayout` 取回 $500$ GREY。此时数组被删，但玩家的索引 mapping 仍存在；后续 `contains(msg.sender)` 继续返回真，而且 `race.payout` 没有置零，因此可以对同一比赛重复领取。Race 合约还保管着 Setup 为第一场比赛投入的 $1000$ GREY，正好可供第二、第三次幽灵领取：

```solidity
setup.race().claimPayout(raceId);
setup.race().claimPayout(raceId);
setup.race().claimPayout(raceId);
```

最终达到解题阈值，获得：

```text
grey{cant_delete_mapping_c60126ce}
```

## 方法总结

Solidity 的 `delete` 不能遍历并清空 mapping。对由数组和 mapping 共同维护的集合直接删除，会破坏 `values` 与 `indexes` 的一致性；之后的 `contains`、`add`、`remove` 都可能产生幽灵成员。本题还暴露了第二个会计问题：领取奖池后没有把 `payout` 置零。正确做法是使用库提供的逐项删除逻辑，并在外部转账前按 Checks-Effects-Interactions 原则清零一次性状态。
