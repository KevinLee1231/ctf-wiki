# N1CTF 2021 - hello

## 题目简述

程序要求输入 32 个十六进制字符，即 16 字节数据。输入先经过一个查表实现的白盒 AES，再由 20 条模 $0x125$ 的线性表达式验证输出。解题分为两步：先解线性方程得到 AES 密文，再从白盒查找表恢复 AES 密钥并解密。

## 解题过程

### 求解最终校验方程

反编译结果中的 20 个判断都形如

$$
\sum_{j=0}^{15}k_{i,j}v_j\equiv b_i\pmod{0x125}.
$$

原 WP 的图片只展示了第一条反编译表达式，没有额外视觉信息；将所有系数直接抄入矩阵即可。由于 $0x125=293$ 是素数，可以在 $\operatorname{GF}(293)$ 上求解这个 20 行、16 列的超定但相容系统：

```python
A = matrix(GF(0x125), coefficients)
b = vector(GF(0x125), targets)
v = A.solve_right(b)
```

程序的字节顺序与反编译变量顺序相反，翻转后得到：

```text
c9 f7 24 d3 1a e0 f1 83 70 18 02 00 11 f3 38 ba
```

也就是白盒 AES 应输出的 16 字节密文。

### 从白盒 T-box 恢复密钥

`func1` 不是普通 AES 轮函数，而是把 SubBytes、AddRoundKey、MixColumns 以及编码组合进多组查找表。它没有额外的外部输入/输出编码保护，属于可直接剥离的 T-box 白盒 AES。

官方解法使用 [CryptoAttacks](https://github.com/GrosQuildu/CryptoAttacks) 中的 `whitebox_aes_sage`：先生成标准 AES 的 Ty 表，将程序中的 T 表与 Ty 表组合，再调用未保护白盒 AES 的密钥恢复函数。

```python
from CryptoAttacks.Block.whitebox_aes_sage import *
from CryptoAttacks.Utils import *

Ty = generate_tyboxes()
composed = compose_T_Ty_boxes(T, Ty)
key_matrix = recover_key_unprotected_wbaes(composed, Ty)
key = bytes(matrix_to_array(key_matrix))
print(key)
```

恢复出的 16 字节密钥为：

```text
NU1Lnu1lnu1lNU1L
```

### 解密得到 flag

用 AES-ECB 解密前一步恢复的密文：

```python
from Crypto.Cipher import AES

key = b"NU1Lnu1lnu1lNU1L"
ciphertext = bytes.fromhex("c9f724d31ae0f1837018020011f338ba")
plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
print(f"n1ctf{{{plaintext.hex()}}}")
```

最终得到：

```text
n1ctf{bc9460b17231c7e374be587427cc3f1a}
```

## 方法总结

题目把“恢复期望密文”和“恢复白盒密钥”分成两个独立障碍。线性校验应先整体矩阵化，避免手工逐式回代；确认查表 AES 没有外部编码后，再使用标准 T-box 分解工具恢复轮密钥。反编译公式截图适合转成系数矩阵文本，不值得作为图片保留。
