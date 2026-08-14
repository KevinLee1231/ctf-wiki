# Transient Heist

## 题目简述

`Setup` 初始化 `USDSEngine`、三组 `WETH/USDC/SafeMoon` 资产，并将 `isSolved` 定义为：

$$
\text{collateralDeposited(player, token0)} > \text{keccak256("YOU NEED SOME BUCKS TO GET FLAG")}
$$

两种 `collateralToken` 均需同时越过该阈值。题面未给直接接口，关键在 `depositCollateralThroughSwap` 与回调参数链路中的状态一致性问题。

## 解题过程

官方脚本 `admin/Exploit/Solve.s.sol` 给出可执行链路，核心在“暂存槽复用 + 回调参数伪造”。

`USDSEngine.depositCollateralThroughSwap(_otherToken,_collateralToken,swapAmount,_collateralDepositAmount)` 中的关键流程是：

1. 把 `swapAmount` 转入借贷合约；
2. 通过 `IBi0sSwapPair.swap(...)` 发起交换；
3. 用 `tstore(1, bi0sSwapPair)` 暂存当前 pair；
4. 等待回调 `bi0sSwapv1Call`；
5. 回调内又用 `tstore(1, tokensSentToUserVault)` 覆盖同一槽位。

在当前仓库的原始 `USDSEngine` 版本里，`depositCollateralThroughSwap` 只对 `_otherToken` 应用 `acceptedToken`，没有验证 `_collateralToken`；Revenge 才对两者分别检查。不过官方预期脚本本身传入的 WETH 与 SafeMoon 都是合法代币，真正让两版都可利用的是回调把槽位 1 从 pair 地址覆盖为攻击者可控整数。

官方 `Exploit.pwn()` 选取 `Create2Deployer` 部署攻击合约，随后执行：

```solidity
uint256 this_addr=uint256(uint160(address(this)));
uint256 requiredAmount = uint256(keccak256("YOU NEED SOME BUCKS TO GET FLAG")) + 1;
usdcEngine.depositCollateralThroughSwap(weth, safemoon, 80000e18, _collateralDepositAmount);
usdcEngine.bi0sSwapv1Call(player, weth, requiredAmount + this_addr, abi.encode(requiredAmount));
usdcEngine.bi0sSwapv1Call(player, safemoon, requiredAmount + this_addr, abi.encode(requiredAmount));
setup.setPlayer(player);
```

第一次交换的实际 `amountOut` 是脚本中的常量 `1207000603499873710129495113646411976443`。脚本令 `_collateralDepositAmount = amountOut - uint160(address(this))`，所以真实回调结束时：

$$
\text{tokensSentToUserVault}=\text{amountOut}-\text{collateralDepositAmount}=\text{uint160(exploit)}.
$$

槽位 1 因而保存攻击合约地址。随后攻击合约手工调用 `bi0sSwapv1Call`，此时 `msg.sender` 正好等于 `tload(1)`；传入 `amountOut = requiredAmount + this_addr`、`collateralDepositAmount = requiredAmount` 又会让差值保持为 `this_addr`，所以同一槽值可以连续复用两次，把 WETH 与 SafeMoon 的虚假抵押都记到 `player` 而不是攻击合约名下。

官方题解同样以 EIP-1153 瞬态槽位复用、CREATE2 固定攻击合约地址、真实回调把槽位 1 改写为该地址，以及两次手工回调伪造抵押为主线；所需等式和调用顺序已在上文完整展开，链接仅用于核对原始说明：[bi0s Pentest 官方题解](https://pentest.bi0s.in/blog/posts/TransientHeist-bi0sCTF2025/)。

## 方法总结

- 本题机制不是“单次参数越界”，而是“瞬态状态污染 + 回调路径重用”。
- 复盘要点：
  - `depositCollateralThroughSwap` 中 `tstore(1, bi0sSwapPair)` 与回调后 `tstore(1,tokensSentToUserVault)` 的覆盖顺序；
  - 在回调里控制 `amountOut - collateralDepositAmount` 为攻击合约地址；
  - 用 `create2` 固定部署地址配合上述恒等关系（`requiredAmount + this_addr`）形成可复用调用。
- 这类题建议优先按 `msg.sender` 与 `transient storage` 生命周期去逐步验证：先确认“单次真实回调”写了什么，再确认“手工再调”是否能在同交易内复用该槽值。
