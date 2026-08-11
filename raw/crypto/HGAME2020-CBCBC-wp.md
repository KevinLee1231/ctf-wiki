# CBCBC

## 题目简述

服务端把 AES-ECB 原语串成两层相互反馈的 CBC 结构，密文格式为 `IV1 || IV2 || C1 || C2 || ...`。用户可以提交任意密文；解密结果不回显，但 PKCS#7 去填充成功时返回 `decryption done`，失败时返回 `sth must be wrong`。这一个比特的差异构成 padding oracle，可以逐块恢复加密的 flag。

## 解题过程

记 AES 的加密和解密为 $E_K$、$D_K$，两条初始链分别为 $A_0=IV_1$、$B_0=IV_2$。第 $i$ 个明文块的加密过程是：

$$
A_i=E_K(P_i\oplus A_{i-1}),
$$

$$
C_i=E_K(A_i\oplus B_{i-1}),\qquad B_i=C_i.
$$

相应的解密状态转移为：

$$
A_i=D_K(C_i)\oplus B_{i-1},
$$

$$
P_i=D_K(A_i)\oplus A_{i-1},\qquad B_i=C_i.
$$

服务端最后对 `IV1 || IV2 || P1 || P2 || ...` 调用 PKCS#7 `unpad`。只要把希望攻击的明文块截成最后一块，响应差异就能判断其末尾是不是合法填充。

关键是找出每个 $P_i$ 中可线性控制的前一状态：

- $P_1$ 直接与 $A_0=IV_1$ 异或，因此修改 `IV1`；
- $P_2$ 与 $A_1=D_K(C_1)\oplus IV_2$ 异或，因此修改 `IV2`；
- $P_3$ 与 $A_2=D_K(C_2)\oplus C_1$ 异或，因此修改 `C1`；
- $P_4$ 与 $A_3=D_K(C_3)\oplus C_2$ 异或，因此修改 `C2`。

以恢复某块最后一个字节为例，对控制块最后一字节异或枚举值 $g$。当 oracle 接受 `0x01` 填充时：

$$
P[-1]\oplus g=1,
$$

所以原字节为 $g\oplus1$。已知最后 $j$ 个字节后，把它们分别异或成值 $j+1$，再枚举倒数第 $j+1$ 个字节即可。下面的核心函数把网络交互抽象为 `oracle(payload)`；它只在收到 `decryption done` 时返回 `True`：

```python
def xor(a, b):
    return bytes(x ^ y for x, y in zip(a, b))

def recover_block(control, build_payload, oracle, known_suffix=b""):
    """恢复由 control 线性影响、且被 build_payload 放在末尾的明文块。"""
    recovered = known_suffix

    for pad_value in range(len(recovered) + 1, 17):
        for guess_delta in range(256):
            mask = bytearray(16)
            mask[-pad_value] = guess_delta

            # 已恢复的末尾字节全部调整为当前 PKCS#7 填充值。
            for offset, plain_byte in enumerate(reversed(recovered), 1):
                mask[-offset] = plain_byte ^ pad_value

            modified = xor(control, mask)
            if oracle(build_payload(modified)):
                recovered = bytes([guess_delta ^ pad_value]) + recovered
                break
        else:
            raise RuntimeError("no valid padding candidate")

    return recovered
```

解析服务端给出的密文后，四次调用分别为：

```python
iv1, iv2 = encrypted[:16], encrypted[16:32]
c1 = encrypted[32:48]
c2 = encrypted[48:64]
c3 = encrypted[64:80]
c4 = encrypted[80:96]

m1 = recover_block(
    iv1,
    lambda changed: changed + iv2 + c1,
    oracle,
)
m2 = recover_block(
    iv2,
    lambda changed: iv1 + changed + c1 + c2,
    oracle,
)
m3 = recover_block(
    c1,
    lambda changed: iv1 + iv2 + changed + c2 + c3,
    oracle,
)

# 已知原消息按 PKCS#7 补了四个 0x04；给出其中一个即可从
# pad_value=2 开始，剩余三个填充字节也会像普通字节一样恢复。
m4 = recover_block(
    c2,
    lambda changed: iv1 + iv2 + c1 + changed + c3 + c4,
    oracle,
    known_suffix=b"\x04",
)

print(m1, m2, m3, m4)
```

逐块结果为：

```text
hgame{I_like_Pad
ding_oracle_atta
ck_6f64ab782042f
3f389f590a2}\x04\x04\x04\x04
```

去掉 PKCS#7 填充后得到：

```text
hgame{I_like_Padding_oracle_attack_6f64ab782042f3f389f590a2}
```

服务端的双层状态转移和完整分块结果由 [HGAME 2020 Crypto Writeup](https://blog.soreatu.com/posts/writeup-for-crypto-problems-in-hgame-2020/) 交叉核对；正文已经给出不依赖外部示意图的公式、控制块映射和恢复代码。

## 方法总结

- 自定义分组模式仍可能继承 CBC 的可篡改性；需要从解密状态转移中找出哪个可控块与目标明文线性异或。
- oracle 不需要回显明文，只要错误类型或响应不同，就足以逐字节判断合法填充。
- 修复方式是使用经过审查的 AEAD 模式，在解密和解析前先验证认证标签，并让所有失败路径保持一致，避免自制级联模式。
