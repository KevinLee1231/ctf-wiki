# gnosis-unsafe

## 题目简述

一个三人 Safe 允许先排队、等待一分钟后再执行交易。所有 owner 都是不可控地址，交易看似还必须通过签名校验。合约固定使用 Solidity 0.8.15，恰好受 calldata tuple ABI 重编码的 head overflow 编译器漏洞影响；该漏洞可让排队阶段检查到的 signer 与写入队列哈希的 signer 不同。

## 解题过程

`queueTransaction()` 先直接读取 `transaction.signer` 并确认它是 owner，随后计算：

```solidity
queueHash = keccak256(abi.encode(transaction, v, r, s));
```

`transaction` 因含有动态 `bytes data` 而是动态 tuple，后面又跟着定长的 `bytes32[3] s`。旧编译器在把最后的静态 calldata 数组重编码到内存时，会错误执行一次 32 字节清零，并覆盖动态尾部的第一个字；这里正好是 `transaction.signer`。

第一步使用真实 owner `address(0x1337)` 通过显式检查，准备一笔让 Safe 调用 GREY `transfer` 的交易，同时把三组签名数组全设为零：

```solidity
transaction = ISafe.Transaction({
    signer: address(0x1337),
    to: address(setup.grey()),
    value: 0,
    data: abi.encodeCall(GREY.transfer, (attacker, 10_000e18))
});
safe.queueTransaction(v, r, s, transaction);
```

ABI 重编码时 signer 被清成零，因此实际入队哈希对应 `signer = address(0)`。等待 veto 期结束后，把本地交易对象的 signer 改为零，再调用 `executeTransaction(..., 0)`。这一次算出的 `queueHash` 与之前一致。

执行阶段对全零的 $(v,r,s)$ 调用 `ecrecover` 会返回零地址，而代码只比较恢复地址是否等于 `transaction.signer`，没有拒绝零地址，因此签名校验也通过。Safe 最终向攻击者转出全部代币，得到：

```text
grey{code_is_not_law_compiler_bug}
```

## 方法总结

漏洞链同时依赖编译器重编码缺陷和应用层未拒绝零地址签名：前者把“通过 owner 检查的值”变成队列中的零值，后者让零值可通过执行校验。Solidity 官方在 0.8.16 修复了该缺陷。审计固定旧编译器的合约时，必须结合版本安全公告检查 ABI 重编码路径；签名验证也应始终显式拒绝 `address(0)`。
