# AES

## 题目简述

题目实现了一个类 AES 的 9 字节分组密码：状态可视为 $3\times3$ 矩阵，所有运算位于 $GF(3^5)$，共执行 4 轮，最后一轮省略列混合。服务允许任意次选择明文加密，提交完整 9 字节原始密钥后返回 flag。

轮数过少且最后一轮没有扩散层，因此可以对 243 个选择明文实施 Square/积分攻击，逐字节恢复第 4 轮子密钥，再逆推密钥扩展。

## 解题过程

令 243 个明文只在第 0 个字节遍历 $GF(3^5)$ 的全部元素，其余 8 个字节固定为 0。经过 3 轮后，每个状态位置在域上的和均为 0。最后一轮为

$$
C_j=\operatorname{Shift}(S(X))_j+K^{(4)}_j.
$$

猜测某个末轮子密钥字节 $k$，对对应密文字节执行一层逆运算。正确猜测满足

$$
\sum_{m=0}^{242}S^{-1}(C_{m,j}-k)=0.
$$

位置置换不会破坏“和为 0”的性质，所以可以分别枚举 9 个位置的 243 种候选值。

密钥扩展每次追加

```python
sks.append(sks[-1] + SBOX[sks[-9]])
```

已知当前 9 元素窗口 $a_1,\ldots,a_9$ 时，前一个被移出的元素为

$$
a_0=S^{-1}(a_9-a_8).
$$

末轮子密钥向前回退 $4\times9=36$ 次，即可得到初始域元素密钥。

完整利用脚本如下，直接复用附件中的 `Cipher.py` 和 `GF.py`：

```python
from itertools import product

from pwn import remote

from Cipher import BlockCipher, INV_SBOX
from GF import GF


HOST = "TARGET"
PORT = 10007


def encrypt_oracle(io, pt):
    io.sendlineafter(b">", b"E")
    io.sendlineafter(b">", pt.hex().encode())
    io.recvuntil(b"result:")
    return bytes.fromhex(io.recvline().strip().decode())


def rewind_key(last_round_key):
    window = [GF(x) for x in last_round_key]
    for _ in range(36):
        previous = INV_SBOX[window[-1] - window[-2]]
        window = [previous] + window[:-1]
    return bytes(x.to_int() for x in window)


def raw_key_aliases(canonical):
    # GF() 只保留 5 个三进制位，因此 243..255 分别与 0..12 同态。
    candidates = [canonical]
    for pos, value in enumerate(canonical):
        if value < 13:
            extra = [
                key[:pos] + bytes([value + 243]) + key[pos + 1:]
                for key in candidates
            ]
            candidates.extend(extra)
    return candidates


io = remote(HOST, PORT)

# 第 0 字节遍历 GF(3^5)，形成一个积分集。
ciphertexts = []
for value in range(243):
    plaintext = bytes([value]) + b"\x00" * 8
    ciphertexts.append(encrypt_oracle(io, plaintext))

# 分别恢复 9 个末轮子密钥字节。
possible = []
for index in range(9):
    candidates = []
    for guess in range(243):
        total = GF(0)
        for ciphertext in ciphertexts:
            total = total + INV_SBOX[GF(ciphertext[index]) - GF(guess)]
        if total == GF(0):
            candidates.append(guess)
    possible.append(candidates)

assert all(possible)

for last_round_key in product(*possible):
    canonical = rewind_key(last_round_key)

    # 用已知的全零明密文对排除积分条件产生的伪候选。
    if BlockCipher(canonical, 4).encrypt(b"\x00" * 9) != ciphertexts[0]:
        continue

    # 加密只依赖域元素，但服务比较的是 urandom 产生的原始字节；
    # 因此必须枚举 0..12 与 243..255 的所有等价表示。
    for raw_key in raw_key_aliases(canonical):
        io.sendlineafter(b">", b"F")
        io.sendlineafter(b">", raw_key.hex().encode())
        response = io.recvline()
        if b"flag" in response.lower():
            print(response.decode().strip())
            raise SystemExit
```

成功提交原始密钥后得到：

```text
0xGame{fad39b71-7a69-4df4-b420-175332e1852b}
```

## 方法总结

本题的关键是“4 轮、末轮无列混合、域大小仅 243”。积分集让末轮每个密钥字节可以独立枚举，复杂度远低于直接搜索 $2^{72}$ 个原始密钥。还要注意 `GF(value)` 会把 243～255 映射为 0～12：积分攻击只能恢复域元素表示，无法区分这些原始字节，必须利用服务允许重复猜测的逻辑枚举所有等价表示。
