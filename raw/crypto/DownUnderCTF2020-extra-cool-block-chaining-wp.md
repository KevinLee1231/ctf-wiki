# DownUnderCTF 2020 - Extra Cool Block Chaining

## 题目简述

服务实现了自定义 AES 分组模式，并公开加密后的 flag。每次连接允许一次最多 16 字节的加密查询和一次最多 16 字节的解密查询。加密先分别计算 $E_K(P_i)$，再与前一个内部输出链接，最后对每个分组异或同一个 IV；PKCS#7 padding 会在输入恰好满块时额外增加一个完整填充分组。

## 解题过程

令内部值为 $U_i$、最终密文为 $C_i$，则实现可写成：

$$
U_0=E_K(P_0),\qquad U_i=E_K(P_i)\oplus U_{i-1},\qquad C_i=U_i\oplus IV.
$$

解密 oracle 对单个输入块 $X$ 返回 $D_K(X\oplus IV)$。因此把 flag 的首块 $C_0$ 直接提交，就能得到 $P_0$。

恢复后续分组前需要得到 IV。提交恰好 16 个 `0x10` 字节时，PKCS#7 会再补一个完全相同的 `0x10` 分组，所以 $P_0=P_1$：

$$
U_1=E_K(P_1)\oplus U_0=0,
$$

第二个输出分组自然就是 $C_1=IV$。

对于 flag 的第 $i$ 个后续分组，构造单块查询：

$$
X=C_{i-1}\oplus IV\oplus C_i.
$$

oracle 内部先异或 IV，实际解密的是 $C_{i-1}\oplus C_i=U_{i-1}\oplus U_i=E_K(P_i)$，因而返回 $P_i$。核心恢复逻辑如下：

```python
from pwn import xor
from Crypto.Util.Padding import unpad

plaintext = first_block_from_direct_decryption

# 用 16 个 0x10 的满块输入触发第二个相同 padding 块。
enc = encryption_query(b"\x10" * 16)
iv = enc[16:32]

for i in range(1, len(flag_ciphertext) // 16):
    previous = flag_ciphertext[(i - 1) * 16:i * 16]
    current = flag_ciphertext[i * 16:(i + 1) * 16]
    crafted = xor(previous, iv, current)
    plaintext += decryption_query(crafted)

print(unpad(plaintext, 16).decode())
```

最终恢复出：

```text
DUCTF{4dD1nG_r4nd0M_4rR0ws_4ND_x0RS_h3r3_4nD_th3R3_U5u4Lly_H3lps_Bu7_n0T_7H1s_t1m3_i7_s33ms!!}
```

## 方法总结

分析自定义分组模式时，应给每个中间量命名并逐块写成代数式，再寻找可抵消项。本题的两个关键点是：满块 PKCS#7 会产生第二个相同分组，从而泄露 IV；已知 IV 后，可把相邻密文异或关系转换成解密 oracle 能直接处理的 $E_K(P_i)$。
