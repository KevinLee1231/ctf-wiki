# bi0sCTF 2024 - T0Y_box

## 题目简述

附件是一个自定义 AES-128 加密程序和一段密文。程序保留了 AES 的轮密钥扩展、ShiftRows、MixColumns 与 AddRoundKey，却在未被调试时把 S-box 初始化为恒等映射 $S(x)=x$。密文的首个明文块还是程序内置的已知前缀 `Tis is a secret:`。

标准 AES 的安全性依赖非线性的 SubBytes；恒等 S-box 使整个加密过程关于明文和密钥变成 GF(2) 上的仿射映射。利用一组基向量测试即可构造“密钥比特到首块密文比特”的 $128\times128$ 线性系统，求出密钥后正常逆运算全部密文。

## 解题过程

### 确认真实 S-box 与反调试分支

构造函数通过 `ptrace` 风格检查设置 `is_debugged`。正常运行时，`generateNumbers()` 递归生成 $0,1,\ldots,255$，并复制到全局 `sbox`，所以

$$
S(x)=x.
$$

若检测到调试器，程序改用一张无规律的伪 S-box，目的是让动态调试结果误导分析。求解不需要在该分支下运行：从源码或静态反汇编确认正常分支的恒等表，然后在本地复现该算法即可。

程序还把一组字节逐位异或 `i + 15`，解出固定前缀 `Tis is a secret:`。其长度恰好为 16 字节，因此附件密文的第一块拥有完整已知明文。

### 将修改版 AES 写成仿射映射

当 S-box 为恒等映射时：

- SubBytes 不改变状态；
- ShiftRows 是比特位置置换；
- MixColumns 是 GF($2^8$) 上的线性变换，也可视为 GF(2) 线性变换；
- AddRoundKey 是异或；
- 密钥扩展中的字节替换同样为恒等映射，轮常数只贡献固定偏移。

所以对固定已知明文 $P$，首块加密可表示为

$$
E_K(P)=AK\oplus b,
$$

其中 $K$ 是 128 位主密钥。先计算零密钥基准

$$
B=E_0(P),
$$

再对每个单位密钥 $e_i$ 计算

$$
v_i=E_{e_i}(P)\oplus B.
$$

$v_i$ 就是矩阵 $A$ 对应的列。已知首块密文 $C_0$ 后，目标向量为

$$
Y=C_0\oplus B,
$$

求解 $AK=Y$ 即可恢复全部密钥位。

官方 Sage solver 的核心流程为：

```python
known = b"Tis is a secret:"
base = modified_aes_encrypt(known, b"\x00" * 16)

columns = []
for bit in range(128):
    unit_key = (1 << bit).to_bytes(16, "big")
    delta = modified_aes_encrypt(known, unit_key) ^ base
    columns.append(to_little_endian_bits(delta))

A = Matrix(GF(2), columns)
y = vector(GF(2), to_little_endian_bits(first_cipher_block ^ base))
key_bits = A.solve_left(y)
key = bits_to_128_bit_integer(key_bits).to_bytes(16, "big")
```

矩阵按行还是按列放置取决于 `to_bits` 的位序；只要生成矩阵和还原整数使用同一位序即可。求出后应重新加密已知前缀，并检查结果严格等于第一块密文。

### 解密剩余内容

用恢复出的主密钥和恒等 S-box 执行修改版 AES 的逆轮，逐块解密附件：

```python
cipher = ModifiedAES(key, sbox=list(range(256)))
plaintext = b"".join(cipher.decrypt(block)
                     for block in split_blocks(ciphertext, 16))
padding = plaintext[-1]
assert plaintext.endswith(bytes([padding]) * padding)
plaintext = plaintext[:-padding]
```

去除 PKCS#7 风格填充后，明文由已知前缀与 flag 组成。

题目 README 提到的线性密码分析背景，是用线性关系近似密码组件；本题比一般线性密码分析更直接：S-box 不是“近似线性”，而是精确的恒等映射，因此只需 129 次本地加密就能完整建立仿射模型。该背景不影响复现，正文已经给出本题实际使用的精确仿射模型，无需依赖外部教程。

## 方法总结

AES 的其余轮操作几乎都是线性的，真正打破整体线性关系的是 S-box。将 S-box 替换为恒等映射后，密钥扩展和整轮加密都退化成仿射变换。利用一块已知明文，零密钥给出常量项，128 个单位密钥给出矩阵列，在线性系统上求解即可恢复主密钥。反调试分支只会替换 S-box，不应把调试状态下的伪表误当成真实算法。
