# feistel 1/2

## 题目简述

两题使用同一个 64 位轮函数和两轮 Feistel 网络，每块 16 字节，末尾再交换左右半块。`feistel 1` 公开密钥，只要求利用结构可逆性解密；`feistel 2` 隐藏密钥，但故意对 flag 连续执行两次 PKCS#7 填充，使最后一块明文固定为 16 个 `0x10`，从而提供一组轮函数的已知输入、输出。

![Feistel 网络每轮的左右半部交换、轮函数异或及逆向恢复关系](<MoeCTF2023-feistel-1-2-wp/feistel-network-inverse.jpg>)

## 解题过程

两题共用的单块函数如下，后面的脚本直接复用它：

```python
def f(value, key):
    value ^= value >> 4
    value ^= value << 5
    value ^= value >> 8
    value ^= key
    value = (value * 1145 + 14) % (1 << 64)
    value = (value * 1919 + 810) % (1 << 64)
    return value * key % (1 << 64)

def enc(block, key_bytes, rounds=2):
    key = int.from_bytes(key_bytes, "big")
    left = int.from_bytes(block[:8], "big")
    right = int.from_bytes(block[8:], "big")
    for _ in range(rounds):
        left, right = right, f(right, key) ^ left
    left, right = right, left
    return left.to_bytes(8, "big") + right.to_bytes(8, "big")
```

### feistel 1：两轮变换是自身的逆

一轮更新为

$$
(L,R)\longmapsto(R,\,L\oplus f(R,K)).
$$

两轮后题目又交换左右半块。解密 Feistel 网络时只需逆序使用轮密钥；本题两轮使用的是同一个密钥，因此密钥序列逆序后不变，整块 `enc` 可以直接作为解密函数。唯一要删掉的是 ECB 包装中的再次填充。

```python
from Crypto.Util.Padding import unpad

cipher = b'\x0b\xa7\xc6J\xf6\x80T\xc6\xfbq\xaa\xd8\xcc\x95\xad[\x1e\'W5\xce\x92Y\xd3\xa0\x1fL\xe8\xe1"^\xad'
key = b"wulidego"

plain = b"".join(
    enc(cipher[i:i + 16], key, 2)
    for i in range(0, len(cipher), 16)
)
print(unpad(plain, 16))
```

结果为：

```text
moectf{M@g1cA1_Encr1tion!!!}
```

### feistel 2：按位提升恢复密钥

设已知末块为 $P=(L_0,R_0)$，对应密文为 $C=(C_L,C_R)$。按题目两轮及末尾交换的定义，有

$$
C_R=L_2=L_0\oplus f(R_0,K),
$$

因此得到一组轮函数约束：

$$
f(R_0,K)=C_R\oplus L_0.
$$

轮函数所有算术最终都模 $2^{64}$。若等式模 $2^{64}$ 成立，它必然模任意 $2^k$ 成立；异或的低 $k$ 位也不受密钥高位影响。于是可先枚举 $K$ 的最低一位，再逐次尝试给候选补一位，并在模 $2^{k+1}$ 下淘汰不满足约束的候选。某些低位约束不能唯一确定密钥，所以要保留所有分支，而不是贪心选择一个比特。

```python
from Crypto.Util.Padding import unpad
from Crypto.Util.number import bytes_to_long, long_to_bytes

cipher = b'B\xf5\xd8gy\x0f\xaf\xc7\xdf\xabn9\xbb\xd0\xe3\x1e0\x9eR\xa9\x1c\xb7\xad\xe5H\x8cC\x07\xd5w9Ms\x03\x06\xec\xb4\x8d\x80\xcb}\xa9\x8a\xcc\xd1W\x82[\xd3\xdc\xb4\x83P\xda5\xac\x9e\xb0)\x98R\x1c\xb3h'

def pre_transform(value):
    value ^= value >> 4
    value ^= value << 5
    value ^= value >> 8
    return value

def matches_mod_power_of_two(prepared, wanted, key, modulus):
    value = prepared ^ key
    value = (value * 1145 + 14) % modulus
    value = (value * 1919 + 810) % modulus
    value = (value * key) % modulus
    return value == wanted % modulus

def recover_keys(known_plain, known_cipher):
    prepared = pre_transform(bytes_to_long(known_plain[8:]))
    wanted = bytes_to_long(known_cipher[8:]) ^ bytes_to_long(known_plain[:8])

    candidates = [0]
    for bit in range(64):
        modulus = 1 << (bit + 1)
        extended = []
        for prefix in candidates:
            for current_bit in (0, 1):
                key = prefix | (current_bit << bit)
                if matches_mod_power_of_two(prepared, wanted, key, modulus):
                    extended.append(key)
        candidates = extended
    return candidates

known_plain = b"\x10" * 16
keys = recover_keys(known_plain, cipher[-16:])

for key_int in keys:
    key = long_to_bytes(key_int)
    plain = b"".join(
        enc(cipher[i:i + 16], key, 2)
        for i in range(0, len(cipher), 16)
    )
    try:
        flag = unpad(unpad(plain, 16), 16)
    except ValueError:
        continue
    if flag.startswith(b"moectf{") and flag.endswith(b"}"):
        print(flag)
```

有效候选给出：

```text
moectf{F_func_1s_n1t_Ve5y_$EcU%e}
```

## 方法总结

Feistel 网络的可逆性不要求轮函数可逆；解密只依赖交换结构和逆序轮密钥。隐藏密钥版本则利用了模 $2^n$ 运算的低位封闭性：已知一块明文后，可把 64 位搜索拆成逐位提升，并用分支集合处理暂时不唯一的低位解。
