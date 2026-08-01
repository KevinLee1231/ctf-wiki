# GlacierCTF2023 - Glacier Spirit

## 题目简述

服务用同一把 16 字节密钥实现自制 Encrypt-and-MAC。加密部分把 `ascon.encrypt(key, N || i, b'', b'')` 的 16 字节输出当作 CTR 密钥流，认证部分则把同一调用当作 CBC-MAC 的压缩函数。服务先给出 flag 的 nonce、密文和 tag，随后允许八次选择明文查询。

## 解题过程

对第 $i$ 个明文块 $M_i$，加密关系为

$$
C_i=M_i\oplus E_K(N\mathbin\|i).
$$

CBC-MAC 从零链值开始。若只提交一个 16 字节块 $X$，返回的 tag 正好是

$$
T=E_K(X\oplus 0)=E_K(X).
$$

因此，读出 flag 使用的 15 字节 nonce $N$ 后，对每个块查询单块消息 `N || i`，响应 tag 就是该块密钥流。服务会为这次查询重新生成加密 nonce，但它与 MAC 输入无关，不需要使用。

```python
def xor(a, b):
    return bytes(x ^ y for x, y in zip(a, b))

flag = b""
for i, block in enumerate(split_blocks(flag_ct), 1):
    chosen = flag_nonce + i.to_bytes(1, "little")
    query(chosen)                 # 提交 chosen.hex()
    _, _, tag = read_response()
    flag += xor(block, tag)

print(flag)
```

解出的开头包含服务自制 padding；去掉前缀填充字节后得到：

```text
gctf{CTr_M0d3_cbc_M4C_ASCON_DeF3AT$_TH3_$p1rIT}
```

## 方法总结

问题不在 ASCON 本身，而在跨用途复用同一原语与密钥：MAC oracle 暴露了 CTR 所需的精确函数值。分析自制 Encrypt-and-MAC 时，应把加密与认证分别写成代数关系，再检查选择消息查询能否让一侧直接计算另一侧的密钥流。
