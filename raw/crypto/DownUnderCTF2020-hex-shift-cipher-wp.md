# DownUnderCTF 2020 - Hex Shift Cipher

## 题目简述

程序先把明文编码成十六进制字符，再用十六字符表 `0123456789abcdef` 的一个随机排列作为密钥。每个密文字符不仅依赖当前明文字符，还依赖前一个明文字符在密钥排列中的位置。题目同时泄露固定前缀 `The secret message is:`，因此可以用已知明密文对恢复排列。

## 解题过程

`add` 函数逐位模 2 相加，实际就是 4 位整数的 XOR。令 $K_i$ 表示密钥排列第 $i$ 个字符，$K(m)$ 表示字符 $m$ 在排列中的下标。除首字符外，加密关系为：

$$
c_i=K_{K(m_i)\oplus K(m_{i-1})}.
$$

对两边取排列下标可得：

$$
K(c_i)\oplus K(m_i)\oplus K(m_{i-1})=0.
$$

把十六个字符各自未知的排列位置看成 $\mathrm{GF}(16)$ 中的变量，每个已知明密文位置都给出一条线性约束。取 16 条约束组成矩阵 $M$，则密钥位置向量 $\mathbf{k}$ 满足 $M\mathbf{k}=0$。枚举右核基的少量线性组合，并筛掉不是 $0\ldots15$ 全排列的向量，即可得到候选密钥。

恢复出排列后，解密与加密使用同一个 XOR 逆运算：

```sage
ALPHABET = "0123456789abcdef"

class Cipher:
    def __init__(self, key):
        self.key = key

    def decrypt(self, ciphertext):
        state = 7
        plaintext = ""
        for char in ciphertext:
            index = self.key.index(char) ^^ state
            plain = self.key[index]
            plaintext += plain
            state = self.key.index(plain)
        return plaintext
```

构造矩阵与枚举核向量的核心代码如下；`known_pt` 是固定前缀的十六进制编码，`known_ct` 是对应长度的密文前缀：

```sage
from itertools import product

def gf16(value):
    bits = [(value >> i) & 1 for i in range(4)]
    return GF(16)(bits)

def equation_matrix(known_pt, known_ct):
    rows = []
    for c, previous, current in zip(known_ct[1:], known_pt, known_pt[1:]):
        row = [GF(16)(0)] * 16
        row[ALPHABET.index(c)] += gf16(1)
        row[ALPHABET.index(previous)] += gf16(1)
        row[ALPHABET.index(current)] += gf16(1)
        rows.append(row)
    return rows

def vector_to_key(vector):
    key = [None] * 16
    for alphabet_index, value in enumerate(vector):
        key[value.integer_representation()] = ALPHABET[alphabet_index]
    return ''.join(key)

known_pt = b"The secret message is:".hex()
known_ct = ciphertext[:len(known_pt)]
rows = equation_matrix(known_pt, known_ct)

for offset in range(1, len(rows) - 16):
    matrix = Matrix(GF(16), rows[offset:offset + 16])
    nullity = matrix.nullity()
    if nullity > 5:
        continue
    basis = matrix.right_kernel_matrix()
    for coefficients in product(range(16), repeat=nullity):
        vector = basis.linear_combination_of_rows([gf16(x) for x in coefficients])
        positions = [x.integer_representation() for x in vector]
        if len(set(positions)) != 16:
            continue
        key = vector_to_key(vector)
        decoded = bytes.fromhex(Cipher(key).decrypt(ciphertext))
        if b"DUCTF{" in decoded:
            print(decoded.decode())
            raise SystemExit
```

得到：

```text
DUCTF{d1d_y0u_Us3_gu3ss1nG_0r_l1n34r_4lg3bRA??}
```

## 方法总结

自定义替换密码若只是在秘密排列的下标上执行线性运算，已知明文往往会把排列恢复问题转成线性约束。本题的识别重点是 `add` 实际为 XOR，以及 `key.index()` 让每个字符都对应一个未知的四位位置；求核后还必须施加“十六个位置互不相同”的排列约束。
