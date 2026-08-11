# DownUnderCTF 2023 fnv Writeup

## 题目简述

服务要求提交任意十六进制字符串，使 64 位 FNV-1 哈希等于 `0x1337133713371337`。FNV 的状态只有 64 位，更新仅包含模 $2^{64}$ 的乘法与单字节异或，并不具备密码学抗原像性。

## 解题过程

状态递推为：

$$
h_{i+1}=\bigl(p h_i\bmod 2^{64}\bigr)\oplus b_i,
$$

其中 $p=0x100000001b3$，初值为 `0xcbf29ce484222325`。官方解法选择 10 个输入字节，用格规约寻找一组很小的中间修正量，再逐字节枚举 $0\ldots255$，把异或带来的非线性约束补回去。

核心 Sage 脚本如下：

```python
TARGET = 0x1337133713371337
h0 = 0xcbf29ce484222325
p = 0x00000100000001B3
modulus = 2**64
n = 10

matrix = Matrix.column(
    [p ** (n - i - 1) for i in range(n)]
    + [-(TARGET - h0 * p**n), modulus]
)
matrix = matrix.augment(
    identity_matrix(n + 1).stack(vector([0] * (n + 1)))
)
weights = diagonal_matrix([2**128] + [2**4] * n + [2**8])
matrix = (matrix * weights).BKZ() / weights

for row in matrix:
    if row[0] == 0 and abs(row[-1]) == 1:
        row *= row[-1]
        corrections = row[1:-1]
        break

answer = []
state = int(h0 * p)
target = int((h0 * p**n + corrections[0] * p**(n - 1)) % modulus)

for i in range(n):
    for byte in range(256):
        value = int((state ^ byte) * p**(n - i - 1) % modulus)
        if value == target:
            answer.append(byte)
            if i < n - 1:
                target = int((target + corrections[i + 1] * p**(n - i - 2)) % modulus)
                state = int(((state ^ byte) * p) % modulus)
            break
    else:
        raise ValueError("byte recovery failed")

print(bytes(answer).hex())
```

把输出的十六进制原像提交给服务，即可得到：

```text
DUCTF{sorry_but_your_cryptographic_hash_function_is_in_another_castle}
```

## 方法总结

FNV 适合哈希表等非对抗场景，不应承担密码学原像安全。固定宽度模乘使整个状态空间很小，字节异或又只有 256 种可能；格规约负责逼近目标关系，逐字节枚举负责满足准确的 XOR 递推。
