# Transient Heist Revenge

## 题目简述

“Revenge”比第一问只改了部分白名单逻辑。`USDSEngine` 新增了 `_otherToken` 与 `_collateralToken` 两套校验，但仍保留了原始回调态管理模式。

`Setup` 的题目入口同样是 `isSolved()` 中两种抵押余额分别过阈值，阈值来自同一段哈希常量。

官方赛题博客与仓库脚本都明确显示核心仍在“暂存槽复用”链路，没有被新校验彻底挡住。

## 解题过程

与第一题相比，Revenge 的 `Setup` 在 `initialize` 里增加了：

```solidity
usdsEngine.changeOtherTokenStatus(address(weth), true);
usdsEngine.changeOtherTokenStatus(address(safemoon), true);
usdsEngine.changeOtherTokenStatus(address(usdc), true);
```

并新增 `accepted_other_tokens`、`acceptedCollateralToken` 两套映射，但 `depositCollateralThroughSwap` 的关键状态轨迹未被消除。

`USDSEngine` 的回调路径仍是：

1. swap 时把 `bi0sSwapPair` 写入 `tstore(1)`；
2. 回调里用 `amountOut` 与 `collateralDepositAmount` 写 `tstore(1,tokensSentToUserVault)`；
3. 之后若在同一交易/上下文继续调用，可借槽位值绕过正常来源检查。

官方脚本 `admin/Exploit/Solve.s.sol` 与上一题一致：

- `Create2Deployer` 先选定攻击合约地址；
- 先做一次正常 `depositCollateralThroughSwap(weth, safemoon, 80000e18, _collateralDepositAmount)`；
- 再手工调用 `bi0sSwapv1Call(player, ..., requiredAmount+this_addr, abi.encode(requiredAmount))` 两次；
- 设定 `player` 并触发 `isSolved`。

关键是第一次真实 swap 结束时把 `amountOut - collateralDepositAmount` 精确安排为 `this_addr`。之后手工回调再令 `amountOut = requiredAmount + this_addr`、解码出的 `collateralDepositAmount = requiredAmount`，差值仍等于攻击合约地址；槽位和 `msg.sender` 因而持续对齐，新增的双白名单检查没有覆盖这个状态机错误。

```solidity
uint256 this_addr=uint256(uint160(address(this)));
uint256 requiredAmount = uint256(keccak256("YOU NEED SOME BUCKS TO GET FLAG")) + 1;
usdcEngine.bi0sSwapv1Call(player, address(weth), requiredAmount + this_addr, abi.encode(requiredAmount));
usdcEngine.bi0sSwapv1Call(player, address(safemoon), requiredAmount + this_addr, abi.encode(requiredAmount));
```

## 方法总结

- 这题的“修复版本”只修了 token 白名单语义，没有修掉 transient 槽位与回调可重入复用的核心机制。
- 解决时应按两次回调验证：
  - 正常回调结束后槽位是否为攻击者可控值；
  - 之后的手工 `bi0sSwapv1Call` 是否会在 `msg.sender == tload(1)` 的路径下成立。
- `Create2Deployer` 与 `admin/Exploit/Vanity-Generator/src/main.rs` 用来寻找能满足 `_collateralDepositAmount` 非负且精确对齐的攻击合约地址；它不是装饰性的部署方式。
- 机制背景可对照 [bi0s Pentest 官方题解](https://pentest.bi0s.in/blog/posts/TransientHeist-bi0sCTF2025/)，但所需的槽位覆盖、地址等式和两次伪回调已经完整写入正文。
