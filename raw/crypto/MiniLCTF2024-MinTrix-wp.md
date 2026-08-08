# miniLCTF 2024 MinTrix Writeup

## 题目简述

题目在 $\mathrm{GF}(p)$ 上实现矩阵版密钥交换。私钥为四组 $A\in K^{99\times66}$、$B\in K^{66\times99}$，公钥仅公开 $P=AB$。双方把

$$\det(A_0^T P_1 B_0^T)$$

作为共享值，四个共享值拼接后直接用作 AES-ECB 密钥。

## 解题过程

### 公钥本身足以替代私钥

对任意一组公钥矩阵 $P=AB$，其秩至多为 66。若能找到任意秩分解

$$P=CD,qquad C\in K^{99\times66},\quad D\in K^{66\times99},$$

则 $(C,D)$ 可充当与原私钥等价的一组因子。原因是行列式只依赖因子乘积：

$$
\det(C^T P_B D^T)
=\det(A_B^T(CD)B_B^T)
=\det(A_B^T P_A B_B^T).
$$

因此不需要恢复原始随机矩阵，只需对 Alice 的每个公钥做秩分解，再代入 Bob 公钥计算共享值。

### Sage 求秩分解并解密

```python
from sage.all import load
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes

pkA, pkB, ct_hex = load("output.sobj")
shared = []

for pa, pb in zip(pkA, pkB):
    assert pa.nrows() == 99 and pa.ncols() == 99
    assert pa.rank() == 66

    C = pa.column_space().basis_matrix().transpose()
    D = C.solve_right(pa)
    assert C.ncols() == 66 and D.nrows() == 66
    assert C * D == pa

    shared.append((C.transpose() * pb * D.transpose()).det())

key = b"".join(long_to_bytes(int(x)) for x in shared)
flag = AES.new(key, AES.MODE_ECB).decrypt(bytes.fromhex(ct_hex))
print(shared)
print(key.hex())
print(flag)
```

本地复现出的四个共享整数为：

```text
[1179557241, 2325357500, 1382323640, 967244040]
```

拼接得到的 16 字节密钥为 `464e9d798a9a23bc526495b839a6f908`，明文为：

```text
miniLCTF{th3re 15_a 1n7r3st1ng tr1ck_of_matr1x!}
```

## 方法总结

协议错误地把“因子分解”当成了秘密，而实际共享值只依赖公开乘积 $AB$。任何满足相同乘积和维度的秩分解都能复现共享密钥。审计矩阵密码协议时，应检查最终运算究竟依赖具体私钥，还是只依赖已公开的线性映射。
