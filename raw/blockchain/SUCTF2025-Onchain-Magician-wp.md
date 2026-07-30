# Onchain_Magician

## 题目简述

题目部署了一个 Solidity `MagicBox` 合约。调用者先用自己的 ECDSA 签名执行 `signIn()`，再提供另一份签名执行 `openBox()`。两次签名都必须由 `ecrecover()` 恢复出同一个 `msg.sender`，但合约又要求两份原始 `(v,r,s)` 的哈希不同。

漏洞在于合约把“签名字节不同”误当成“签名语义不同”。secp256k1 上的 ECDSA 签名具有可延展性：一个有效签名可以直接变换成另一个仍可恢复出同一地址的签名。

## 解题过程

合约签名的消息不是任意字符串，而是：

```solidity
keccak256(
    abi.encodePacked(
        "I want to open the magic box",
        msg.sender,
        address(this),
        block.chainid
    )
)
```

因此必须从题目实例读取 `getMessageHash(account)`，并让持有该 `account` 私钥的账户对这个 32 字节摘要签名。

设 secp256k1 的阶为 $n$。若 $(r,s)$ 是一份有效 ECDSA 签名，则 $(r,n-s)$ 也是同一消息、同一公钥下的有效签名；恢复公钥时还要翻转恢复编号：

```python
SECP256K1_N = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141

new_r = r
new_s = SECP256K1_N - s
new_v = 28 if v == 27 else 27
```

这是因为验证方程中使用 $s^{-1}$，把 $s$ 换成 $n-s\equiv-s\pmod n$ 会把对应椭圆曲线点取反；翻转 `v` 后，`ecrecover()` 仍恢复出原地址。与此同时，

```solidity
keccak256(abi.encodePacked(v, r, s))
```

和变换后的签名哈希不同，正好绕过 `alreadyUsedSignatureHash`。

完整调用顺序如下：

```python
message_hash = contract.functions.getMessageHash(account.address).call()
signature1 = account.signHash(message_hash)

contract.functions.signIn(
    (signature1.v, signature1.r, signature1.s)
).transact({"from": account.address})

signature2 = (
    28 if signature1.v == 27 else 27,
    signature1.r,
    SECP256K1_N - signature1.s,
)
contract.functions.openBox(signature2).transact({"from": account.address})

assert contract.functions.isSolved().call() is True
```

实际库的字段类型可能需要把 `r`、`s` 转成 32 字节，但变换关系不变。第一次交易登记原签名，第二次交易提交等价签名；`signer == msg.sender`、签名哈希不同和调用者身份三个检查会同时通过，最终 `isSolved()` 变为 `true`。

题目仓库给出的结果为：

```text
SUCTF{C0n9r4ts!Y0u're_An_0ut5taNd1ng_OnchA1n_Ma9ic1an.}
```

## 方法总结

本题的关键不是伪造他人的签名，而是利用 ECDSA 的等价表示。只比较原始签名字节无法实现可靠的防重放：合约应先强制 `s` 落在低半区并规范化 `v`，或直接记录消息、nonce 和 signer 等业务语义。即使现代交易签名通常执行 low-`s` 规范化，Solidity 的原生 `ecrecover()` 仍需要合约显式做范围检查，不能把底层规范当作自动防护。
