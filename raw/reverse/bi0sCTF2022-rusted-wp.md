# bi0sCTF 2022 Rusted Writeup

## 题目简述

Rusted 是一道 Rust ELF 逆向题。程序先使用固定密钥和 IV 对用户输入执行 SM4-CBC 加密，再把前 32 字节密文拆成八个大端 32 位整数。随后，它在 RWX 内存中即时拼出一段 x86-64 校验函数，用若干减法和异或关系检查这些整数。

题目提示“每次减法的结果都应为 0”，因此无需逆向求解 SM4：先把 JIT 中的比较关系交给 Z3，恢复目标密文，再使用二进制中的固定 SM4 参数做 CBC 解密即可。

## 解题过程

### 识别 SM4-CBC 参数

Rust 源码内置了完整 SM4 S 盒、轮常量和 CBC 封装。`enc` 函数直接给出：

```text
key = UBHPjBKlsQ2TuNSk
iv  = 7y0M69TQScm7mfXv
```

输入经过 PKCS#7 填充后加密。JIT 生成器把 `inp[0:32]` 每四字节逆序写入 x86 小端立即数，等价于将密文按大端序解释成 $e_0,\ldots,e_7$。

### 从 JIT 恢复约束

结合寄存器操作和题目提示，每个减法前的表达式都应等于对应常量。将 32 位环上的关系写成等式：

$$
\begin{aligned}
e_0 &= \mathtt{0x93A3F3CD},\\
e_1 &= \mathtt{0x1337BEEF}\oplus\mathtt{0x33AEF5CB},\\
e_2\oplus e_4 &= \mathtt{0x550D68CE},\\
e_2\oplus e_3 &= \mathtt{0x5F9751EB},\\
e_2\oplus e_3\oplus e_4 &= \mathtt{0x74180051},\\
e_7\oplus e_5 &= \mathtt{0x04AA34A4},\\
e_7\oplus e_6 &= \mathtt{0x2C786553},\\
e_5\oplus e_6\oplus e_7 &= \mathtt{0x3E07994C}.
\end{aligned}
$$

第一条在原求解脚本中写成 `-0x6C5C0C33`。因为 JIT 只保留低 32 位，它与无符号的 `0x93A3F3CD` 是同一个位模式。

### 用 Z3 恢复密文并解密

以下代码完整保留了 32 位位向量语义，并按大端序拼回 32 字节密文：

```python
from z3 import BitVec, Solver, sat
from gmssl.sm4 import CryptSM4, SM4_DECRYPT

e = [BitVec(f"e{i}", 32) for i in range(8)]
s = Solver()

s.add(e[0] == 0x93A3F3CD)
s.add(e[1] == (0x1337BEEF ^ 0x33AEF5CB))
s.add((e[2] ^ e[4]) == 0x550D68CE)
s.add((e[2] ^ e[3]) == 0x5F9751EB)
s.add((e[2] ^ e[3] ^ e[4]) == 0x74180051)
s.add((e[7] ^ e[5]) == 0x04AA34A4)
s.add((e[7] ^ e[6]) == 0x2C786553)
s.add((e[5] ^ e[6] ^ e[7]) == 0x3E07994C)

assert s.check() == sat
model = s.model()
ciphertext = b"".join(
    model.eval(word).as_long().to_bytes(4, "big") for word in e
)

sm4 = CryptSM4()
sm4.set_key(b"UBHPjBKlsQ2TuNSk", SM4_DECRYPT)
plain = sm4.crypt_cbc(b"7y0M69TQScm7mfXv", ciphertext)
print(plain.decode())
```

所用 `gmssl` 版本的 CBC 解密接口会移除 PKCS#7 填充；若换用只返回原始明文块的实现，应自行验证并删除末尾填充。输出为：

```text
bi0s{jitrustyjeRPUGEbTa}
```

### 回代验证

仅看到可打印字符串还不足以证明求解正确。将恢复出的 32 字节重新按大端分组，八条 Z3 关系应全部成立；再用相同密钥和 IV 对明文执行 SM4-CBC 加密，结果也应与恢复的密文逐字节相同。这同时验证了 JIT 字节序和 CBC 参数。

## 方法总结

这道题把密码算法和 JIT 逆向串在一起，但决定性障碍是恢复运行时校验关系。不要尝试从已知密文反推 SM4 密钥：密钥和 IV 已明文存在于程序中，未知量只是被 JIT 约束的目标密文。

处理即时生成代码时，最有效的做法是先把立即数、寄存器和每次比较的中间值提升为固定宽度位向量。特别要保留 32 位回绕与大端分组语义，否则负常量 `-0x6C5C0C33` 很容易被误判为与 `0x93A3F3CD` 不同。
