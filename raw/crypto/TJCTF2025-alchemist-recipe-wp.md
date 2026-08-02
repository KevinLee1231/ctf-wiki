# alchemist-recipe

## 题目简述

题目给出一段经过自定义分组算法加密的十六进制数据和完整加密代码。代码中的变量名虽然被故意改成无意义单词，但算法本身没有隐藏密钥：固定字符串 `AurumPotabileEtChymicumSecretum` 经 SHA-256 后，依次派生 8 字节置换材料、16 字节异或材料和 S 盒构造材料。明文使用 8 字节分组和类似 PKCS#7 的填充，每组依次经过 S 盒替换、异或和字节置换。

## 解题过程

变量名不会影响数据流。先按加密代码重建派生过程：

1. `SHA256(seed)` 的前 8 字节决定块内位置排列；
2. 接下来的 16 字节作为循环异或密钥；
3. 最后 8 字节反复驱动交换，生成一个 0 到 255 的置换 S 盒；
4. 加密顺序是“代换 → 异或 → 置换”，因此解密必须按“逆置换 → 异或 → 逆 S 盒”执行。

下面的脚本保留了完整的逆变换：

```python
import hashlib

SEED = "AurumPotabileEtChymicumSecretum"
BS = 8

digest = hashlib.sha256(SEED.encode()).digest()
perm_key = list(digest[:BS])
xor_key = digest[BS:BS + 16]

sbox = list(range(256))
pos = 0
for _ in range(256):
    for value in digest[BS + 16:]:
        other = (pos + value) % 256
        sbox[pos], sbox[other] = sbox[other], sbox[pos]
        pos = (pos + 1) % 256

inv_sbox = [0] * 256
for i, value in enumerate(sbox):
    inv_sbox[value] = i

order = [old_index for _, old_index in sorted(
    (perm_key[i], i) for i in range(BS)
)]

def decrypt_block(block: bytes) -> bytes:
    unpermuted = [0] * BS
    for new_index, old_index in enumerate(order):
        unpermuted[old_index] = block[new_index]
    unxored = bytes(
        value ^ xor_key[i % len(xor_key)]
        for i, value in enumerate(unpermuted)
    )
    return bytes(inv_sbox[value] for value in unxored)

with open("encrypted.txt", "r", encoding="utf-8") as f:
    ciphertext = bytes.fromhex(f.read().strip())

plaintext = b"".join(
    decrypt_block(ciphertext[i:i + BS])
    for i in range(0, len(ciphertext), BS)
)
pad_len = plaintext[-1]
assert plaintext[-pad_len:] == bytes([pad_len]) * pad_len
print(plaintext[:-pad_len].decode())
```

在仓库样本上运行后得到：

```text
tjctf{thank_you_for_making_me_normal_again_yay}
```

## 方法总结

- 核心技巧：忽略迷惑性命名，沿数据流逐层构造逆运算。
- 识别信号：固定 seed、可重建的 S 盒、纯置换和 XOR 都是可逆变换，不构成真正的密钥保护。
- 复用要点：复合自定义密码应先准确写出正向顺序，再严格按相反顺序求逆；置换的下标方向尤其容易写反。
