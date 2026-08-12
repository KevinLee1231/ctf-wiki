# DownUnderCTF 2020 - ceebc

## 题目简述

服务把单个 16 字节消息送入 AES-CBC，并把唯一密文块当作 MAC。签名格式为 `tag || iv`，验证端还会从用户提交的签名中取出 IV。服务公开消息 `cashcashcashcash` 的合法签名，目标是伪造 `flagflagflagflag` 的签名。

## 解题过程

对单个分组，服务计算：

$$
T=E_K(M\oplus IV).
$$

真正的 CBC-MAC 应使用固定 IV；这里的 IV 不但随机，还随签名一起交给攻击者。已知合法三元组 $(M,IV,T)$ 后，选择目标消息 $M'$，只需构造：

$$
IV'=M\oplus IV\oplus M'.
$$

于是：

$$
M'\oplus IV'=M\oplus IV,
$$

验证时得到的 AES 输入完全相同，原标签 $T$ 可以直接复用。脚本只需要解析服务给出的 32 字节签名，把前 16 字节保留为标签、后 16 字节替换为伪造 IV：

```python
def xor_bytes(a, b):
    return bytes(x ^ y for x, y in zip(a, b))

known_message = b"cashcashcashcash"
target_message = b"flagflagflagflag"

# known_sig 是服务公开的 32 字节 tag || iv
known_tag = known_sig[:16]
known_iv = known_sig[16:]

forged_iv = xor_bytes(xor_bytes(known_message, known_iv), target_message)
forged_signature = known_tag + forged_iv

print(target_message.decode())
print(forged_signature.hex())
```

把目标消息和伪造签名提交给服务即可通过验证并得到：

```text
DUCTF{WAT_I_THOUGHT_IV_IZ_G00D}
```

## 方法总结

随机 IV 对 CBC 加密很重要，但 CBC-MAC 的安全条件不同：IV 必须固定且不能由攻击者控制。看到单分组 CBC-MAC、签名携带 IV、验证端信任提交 IV 时，应立即检查能否通过调整 IV 保持 $M\oplus IV$ 不变，从而复用已有标签。
