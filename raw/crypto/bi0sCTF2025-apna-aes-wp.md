# APNA-AES

## 题目简述

这道题分成两个连续阶段。第一阶段是自定义双状态 AES 模式的 padding oracle，恢复出的明文给出 `nxt.7z` 的密码；解开压缩包后，第二阶段要求从一组 secp256k1 公钥与 ECDSA 签名中恢复私钥，并把该私钥写成 flag。

仓库中的证据链是完整的：`admin/src/chall.py` 与 `admin/exploit/solve.py` 对应 AES 阶段，`Handout/nxt/message.py`、`Handout/nxt/out.txt` 与 `admin/exploit/solve_ecdsa.py` 对应 ECDSA 阶段。第二份脚本不是无关附件，而是拿到最终 flag 的必要步骤。

## 解题过程

AES 阶段对第 $i$ 个 16 字节块执行：

$$
B_i=P_i\oplus S^{(1)}_i,\qquad C_i=E_k(B_i)\oplus S^{(2)}_i,
$$

随后更新 $S^{(1)}_{i+1}=E_k(B_i)$、$S^{(2)}_{i+1}=B_i$。解密时先计算

$$
D_i=D_k(C_i\oplus S^{(2)}_i),\qquad P_i=D_i\oplus S^{(1)}_i,
$$

并把下一轮状态更新为 $S^{(1)}_{i+1}=C_i\oplus S^{(2)}_i$、$S^{(2)}_{i+1}=D_i$。服务端只返回 `Valid padding` 或 `Invalid padding`，因此可以控制 `IV1`，逐字节恢复 $D_i$。

官方 `solve.py` 对每个密文块先消去当前 `state2`：

```python
dec = single_block_attack(xor(ct_block, block_iv2), b"\x00" * 16)
pt = xor(block_iv1, dec)
block_iv1 = xor(ct_block, block_iv2)
block_iv2 = bytes(dec)
```

`single_block_attack` 从填充长度 1 递增到 16，枚举目标字节并根据 oracle 响应构造合法 PKCS#7 尾部。对 padding 1 还额外翻转倒数第二字节，以排除原密文恰好已有合法填充造成的假阳性。

完整明文是：

```text
Password : sha256(b"Kj+3$98Fv!mL^pR&2xTz@bUe*1qYz-04WFng37%Za=").hexdigest()
```

按该表达式计算得到压缩包密码：

```text
83ed0e4365a128e093e3dab6a27da318f99d490610f9dea318fde4f639f182d8
```

解包后的 `message.py` 生成 secp256k1 私钥 $d$，但故意令 ECDSA nonce 为

$$
k=2^{128}h_H+d_H,
$$

其中 $h_H=\lfloor h/2^{128}\rfloor$ 是消息哈希的高 128 位，$d_H=\lfloor d/2^{128}\rfloor$ 是私钥高 128 位。再写 $d=2^{128}d_H+d_L$，代入 ECDSA 关系

$$
sk\equiv h+rd\pmod n
$$

可得到一个同时约束 $d_H$、$d_L$ 的低维格问题。官方脚本构造：

```sage
X128 = 1 << 128
hmsb = h >> 128
A = X128 - s * inverse_mod(r, n)
b = (h - X128 * s * hmsb) * inverse_mod(r, n)

M = Matrix(ZZ, [
    [n, 0, 0],
    [-A, 1, 0],
    [b, 0, -X128],
])
```

对 `M` 做 LLL 后，筛选第三坐标绝对值等于 $2^{128}$ 的短向量，即可读出候选 $d_H,d_L$，再用 $dG=Q$ 和 $x(kG)\bmod n=r$ 双重校验。最终恢复的私钥十六进制为：

```text
0eb1b6bbaaa68f2587b0f38fe017827e3aacb58f46d12eca8f2276c9fc8d50bf
```

因此 flag 为：

```text
bi0sCTF{0eb1b6bbaaa68f2587b0f38fe017827e3aacb58f46d12eca8f2276c9fc8d50bf}
```

本次没有启动远端 oracle，也没有重新运行 Sage；AES 状态变换、压缩包密码、格构造和最终私钥均与仓库源码、官方脚本及 README 交叉核对。

## 方法总结

第一阶段的关键不是破解 AES 密钥，而是把双状态模式化为“可控 `IV1` 与单块 AES 解密结果异或”的 padding oracle。逐块恢复时必须同步维护 `block_iv1` 和 `block_iv2`，否则从第二块起会全部错位。

第二阶段的关键是 nonce 低 128 位直接等于私钥高 128 位。把私钥拆成高、低两半并代入签名方程后，未知量落进三维格，LLL 给出的短向量再由公钥和签名做确定性筛选。两阶段缺一不可，最终 flag 来自 ECDSA 私钥，而不是第一阶段恢复的压缩包密码。
