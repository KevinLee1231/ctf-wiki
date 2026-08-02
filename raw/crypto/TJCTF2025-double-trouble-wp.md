# double-trouble

## 题目简述

题目使用两层 AES-ECB：先以 `k1 || k2` 加密，再以 `k4 || k3` 加密。`k1` 和 `k3` 都来自每次重新以 42 初始化的局部 PRNG，因此二者相同且完全可预测；`k2`、`k4` 各有 8 字节，但每个字节只从 4 个已知值中选择，搜索空间各为 $4^8=2^{16}$。输出同时给出已知明文 `example` 的双层密文和 flag 密文。

## 解题过程

直接枚举两段弱密钥需要 $2^{32}$ 次组合。双重加密满足

$$E_{k_1\parallel k_2}(P)=D_{k_4\parallel k_3}(C),$$

因此可使用中间相遇攻击：枚举所有 `k2`，保存第一层加密结果；再枚举所有 `k4`，解开已知密文的第二层并查表。总工作量约为 $2\cdot2^{16}$ 次 AES 操作。

```python
import itertools
import random
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad

rng = random.Random(42)
k1 = rng.randbytes(8)
choices = list(rng.randbytes(6))[:4]
k3 = k1

with open("out.txt", "r", encoding="utf-8") as f:
    known_ct = bytes.fromhex(f.readline().strip())
    flag_ct = bytes.fromhex(f.readline().strip())

known_pt = b"example"
middle = {}
for guess in itertools.product(choices, repeat=8):
    k2_guess = bytes(guess)
    key = k1 + k2_guess
    value = AES.new(key, AES.MODE_ECB).encrypt(pad(known_pt, 16))
    middle[value] = k2_guess

for guess in itertools.product(choices, repeat=8):
    k4_guess = bytes(guess)
    key = k4_guess + k3
    value = AES.new(key, AES.MODE_ECB).decrypt(known_ct)
    if value in middle:
        k2 = middle[value]
        k4 = k4_guess
        break
else:
    raise RuntimeError("no key pair found")

stage1 = AES.new(k4 + k3, AES.MODE_ECB).decrypt(flag_ct)
plaintext = AES.new(k1 + k2, AES.MODE_ECB).decrypt(stage1)
print(unpad(plaintext, 16).decode())
```

实测约十余秒即可恢复：

```text
tjctf{m33t_in_th3_middl3}
```

## 方法总结

- 核心技巧：对两层独立弱密钥加密使用 meet-in-the-middle，而不是枚举笛卡尔积。
- 识别信号：已知明密文对、可分开的两层加密，以及每层独立的小密钥空间。
- 复用要点：中间值必须保持与真实加密完全相同的 padding；找到候选后还应解密 flag 并检查 PKCS#7 填充，排除碰撞。
