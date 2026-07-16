# LFSR

## 题目简述

题目使用 128 位 LFSR。每个输出位都是当前 128 位状态与未知 128 位 `mask` 的逐位与后异或，即 $GF(2)$ 上的内积；状态左移并追加该输出。程序连续泄露 256 个输出位（`random1` 与 `random2`），随后直接把 `mask` 作为 AES-128-ECB 密钥。

## 解题过程

已知连续 256 位输出后，第 $i$ 次计算使用的状态就是泄露比特串中的长度 128 滑动窗口。于是前 128 个窗口组成矩阵 $A$，后 128 个输出组成向量 $b$，满足：

$$
A\cdot mask=b\quad\text{over }GF(2)
$$

在 Sage 中解线性方程即可恢复全部 mask 位：

```sage
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

random1 = 79262982171792651683253726993186021794
random2 = 121389030069245976625592065270667430301
ciphertext = b'\xb9WE<\x8bC\xab\x92J7\xa9\xe6\xe8\xd8\x93D\xcc\xac\xfdvfZ}C\xe6\xd8;\xf7\x18\xbauz`\xb9\xe0\xe6\xc6\xae\x00\xfb\x96%;k{Ph\xfa'

def bits(value):
    return [int(x) for x in bin(value)[2:].zfill(128)]

first = bits(random1)
second = bits(random2)
stream = first + second
A = Matrix(GF(2), [stream[i:i + 128] for i in range(128)])
b = vector(GF(2), second)
solution = A.solve_right(b)

mask = sum(ZZ(solution[i]) << (127 - i) for i in range(128))
key = int(mask).to_bytes(16, 'big')
plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
print(unpad(plaintext, 16))
```

## 方法总结

- 核心技巧：把 LFSR 反馈函数写成 $GF(2)$ 线性方程组，从连续输出恢复反馈掩码。
- 识别信号：输出由状态位与固定 mask 的 XOR/AND 组成，并泄露至少约两倍寄存器长度的连续比特。
- 复用要点：构造矩阵时要保持源码的位序和移位方向；恢复整数后再按题目使用的大端序生成 AES key。
