# DeepSea Finance

## 题目简述

`Setup` 在代理金库中放入 10 WBTC，目标是让该余额归零。常规的存款、借款、清算和闪电贷路径都受到抵押率或归还检查限制；`emergencyWithdraw` 能一次转走资产，但仅 guardian 可调用。预期漏洞来自特定编译配置 `solc 0.8.29 + via_ir + Cancun` 对 transient storage 变量 `delete` 的错误代码生成。

源码表面上清除的是瞬态变量 `_epochOperator`，编译结果却对普通 storage 的 slot 1 执行清零，而该槽位正好保存 `governor`。清空初始化锁后，攻击者可以重新初始化金库、把自己设为 guardian，再提走全部 WBTC。

## 解题过程

### 1. 从资产出口反推所需权限

逐个检查 WBTC 转出函数：普通 `withdraw` 需要存款，`borrow` 受 75% LTV 限制，`flashLoan` 必须归还，跨链结算路径也有 route 校验。唯一可以直接清空余额的是：

```solidity
emergencyWithdraw(token, recipient, amount)
```

它要求调用者是 guardian。`initialize` 可以设置 guardian，但前提是 `governor == address(0)`，所以问题转化为“如何把 governor 清零”。

### 2. 验证 transient storage 编译器缺陷

奖励结算链为 `claimRewards` → `_settleRewardEpoch` → `_finalizeRewardEpoch`，关键代码是：

```solidity
function _finalizeRewardEpoch(address operator) internal {
    _epochOperator = operator;
    delete _epochOperator;
}
```

`_epochOperator` 被声明为 `transient`。第一行应生成 `TSTORE`，第二行也应清除 transient slot；但对本题锁定的编译器和 IR 配置检查优化后 IR：

```bash
forge inspect src/vault/DeepSeaVault.sol:DeepSeaVault irOptimized
```

可以看到赋值使用 transient write，而 `delete` 生成了普通 `storage_set_to_zero_address(0x01, 0)`。storage layout 中 slot 1 是 `DeepSeaVault.governor`，因此任何人调用一次 `claimRewards` 都会把 governor 写成零，而且不要求实际存在待领奖励。

### 3. 重新初始化并提走资产

攻击合约按以下顺序执行：

```solidity
vault.claimRewards(wbtc); // 错误 SSTORE，清空 governor

address[] memory guardians = new address[](1);
guardians[0] = address(this);
vault.initialize(address(oracle), address(usdc), guardians);

vault.emergencyWithdraw(
    wbtc,
    player,
    IERC20(wbtc).balanceOf(address(vault))
);
```

重新初始化后攻击合约成为 guardian，第三步直接转出代理金库的全部 WBTC，`isSolved()` 返回 `true`。官方预期利用位于 `exp/Exploit.sol`。

源码还存在一个非预期的经济漏洞：奖励从金库余额直接转出，而用户可以把奖励再次 `deposit`，让 `positions[user][token].deposited` 在“领取—存入”循环中不断膨胀；借款检查信任该记账值，最终也能获得足够抵押额度借空 WBTC。该路线证明奖励会计同样有问题，但步骤更长，也不是题目用 transient storage 配置要考查的主线。

## 方法总结

本题提醒审计不能止于 Solidity 源码语义。遇到锁定编译器版本、`via_ir` 和新 EVM 特性时，应把生成 IR/字节码也纳入证据链，并将异常写操作映射回实际 storage layout。利用链本身很短，但每一步都应验证：`claimRewards` 后 governor 确实为零，重新初始化后 guardian 集合确实受控，最后再调用资产出口。经济漏洞则说明即便预期编译器缺陷被修复，余额与会计份额分离的循环仍需单独审计。
