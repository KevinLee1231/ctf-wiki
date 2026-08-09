# GatekCrash

## 题目简述

题目实现了一个简化的 ERC-4337 `EntryPoint`。管理员账户持有 10 ETH，目标是把余额清零。`handleOps` 先验证整批 `UserOperation`，再统一执行；验证 paymaster 时会设置 `_inPaymasterValidation`。问题在于两个本应属于管理面的接口只检查这个全局阶段标志，却不检查调用者身份：任意恶意 paymaster 都能在回调中把账户加入免签白名单，甚至替换账户验证模块。

源码复核还揭示了官方题解中的一个不准确之处：`owner` 与 `uint48 validationModuleFlag` 共用 slot 0，但后面的 `address validationModule` 无法装入剩余 6 字节，实际位于 slot 1。更重要的是，清空管理员余额只需要白名单漏洞，恶意模块与 `sstore(0,0)` 并非必要条件。

## 解题过程

### 1. 找到跨 UserOperation 的状态污染

`EntryPoint` 在 paymaster 验证前后这样维护标志：

```solidity
_inPaymasterValidation = true;
IPaymaster(paymaster).validatePaymasterUserOp(op, opHash, 0);
_inPaymasterValidation = false;
```

而白名单接口只要求该标志为真：

```solidity
function addToPreApproved(address sender) external {
    require(_inPaymasterValidation, "only during paymaster validation");
    preApprovedSenders[sender] = true;
}
```

因此攻击者部署 paymaster，在自己的操作被验证时调用：

```solidity
entryPoint.addToPreApproved(address(adminAccount));
```

该状态会立即生效，并影响同一批次中随后验证的管理员操作。

### 2. 构造两条操作

第一条 `Op[0]` 的 sender 是攻击者账户，使用攻击者私钥正常签名，并在 `paymasterAndData` 前 20 字节放入恶意 paymaster 地址。它的作用是在验证回调中污染白名单。

第二条 `Op[1]` 的 sender 是管理员账户，nonce 取管理员当前 nonce，`callData` 编码为：

```solidity
adminAccount.execute(attacker, 10 ether, bytes(""));
```

签名只需满足 65 字节格式，因为管理员账户验证逻辑会先检查：

```solidity
if (entryPoint.preApprovedSenders(address(this))) {
    require(userOp.nonce == nonce, "invalid nonce");
    nonce++;
    return 0;
}
```

白名单分支在 `_validateSignature` 之前返回，故无需管理员私钥。

调用 `handleOps([Op[0], Op[1]], beneficiary)` 后，验证阶段先由 `Op[0]` 把管理员加入白名单，再让 `Op[1]` 免签通过；执行阶段由 EntryPoint 调用管理员账户的 `execute`，将 10 ETH 转给攻击者，满足 `isSolved()`。

### 3. 官方模块链的实际作用

官方利用还在 paymaster 回调中调用 `adminUpdateModule(admin, maliciousModule)`。管理员验证第二条操作时会 `delegatecall` 该模块，模块执行 `sstore(0,0)`，清空 slot 0 中的 owner 和 module flag。调用返回后，`validationModuleFlag += 1` 会把 flag 重新置为 1，但 owner 仍为零；slot 1 中的恶意模块地址不会被这次写入清除。

这条链能进一步破坏管理员账户所有权，但对白名单已经提供的免签转账并非必要。最小解应优先保留前两步，模块链可作为额外影响说明，而不应被误写成唯一解法。

## 方法总结

本题的核心是批处理验证期间的共享可变状态。阶段布尔值不能代替调用者授权，更不能让一个 paymaster 修改其他 sender 的认证状态。分析账户抽象题时应沿着“本次回调能写什么、写入何时生效、会影响批次中的谁”检查，而不是只审单条 UserOperation。对 storage packing 也应以编译器 layout 或实际槽位读取为准；官方 WP 的槽位描述与 Solidity 打包规则不符，源码足以证明最小利用根本不依赖该错误描述。
