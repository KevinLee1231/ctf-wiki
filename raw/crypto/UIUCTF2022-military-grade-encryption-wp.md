# Military Grade Encryption

## 题目简述

题目提供一个自制加密网站，并给出由该算法生成的 `flag.enc`。用户输入最多六位 PIN 和所谓的密钥长度；程序把补零后的 PIN 连续做 1000 次 MD5，将结果解释为 128 位整数，再按 $K_i=(i+1)K\bmod2^{128}$ 生成多组 AES-ECB 密钥，循环加密各个 16 字节明文块。

虽然界面使用了“2048 位”“军用级”等措辞，实际密钥空间仍只有六位十进制 PIN 的约 $10^6$ 种可能。重复哈希和派生出很多 AES 密钥都不会增加原始口令的熵。

## 解题过程

核心加密逻辑可以简化为：

```python
key = pin
for _ in range(1000):
    key = md5(key).digest()

K = bytes_to_long(key)
keys = [((i + 1) * K) % 2**128 for i in range(key_size // 16)]
```

每个候选 PIN 都能确定全部 AES 密钥，因此直接枚举 `000000` 到 `999999`，按原算法对 `flag.enc` 解密即可。flag 格式是可靠的低成本判据：正确明文应以 `uiuctf{` 开头，并具有合法 PKCS#7 填充。

```python
from base64 import b64decode
from hashlib import md5
from itertools import cycle
from Crypto.Cipher import AES
from Crypto.Util.number import bytes_to_long, long_to_bytes
from Crypto.Util.Padding import unpad

def derive(pin):
    key = pin
    for _ in range(1000):
        key = md5(key).digest()
    return bytes_to_long(key)

def decrypt(ciphertext, pin, key_size=2048):
    base = derive(pin)
    ciphers = []
    for i in range(key_size // 16):
        value = ((i + 1) * base) % 2**128
        raw = long_to_bytes(value).rjust(16, b"\x00")
        ciphers.append(AES.new(raw, AES.MODE_ECB))

    blocks = [ciphertext[i:i + 16] for i in range(0, len(ciphertext), 16)]
    padded = b"".join(c.decrypt(b) for b, c in zip(blocks, cycle(ciphers)))
    return unpad(padded, 16)

ct = b64decode(open("flag.enc", "rb").read())
for number in range(1_000_000):
    pin = f"{number:06d}".encode()
    try:
        pt = decrypt(ct, pin)
    except ValueError:
        continue
    if pt.startswith(b"uiuctf{") and pt.endswith(b"}"):
        print(pin, pt)
        break
```

官方 solver 使用相同枚举思路，最终恢复：

```text
uiuctf{n0t_eNou6h_3ntr0py_4_H4ndr0113d_Crypto}
```

## 方法总结

- 核心技巧：按真实输入熵评估自制 KDF，对六位 PIN 做离线穷举，并用已知 flag 格式验证解密结果。
- 识别信号：固定小字符集口令、无盐确定性哈希、ECB 和“通过增加轮数或派生密钥数量获得更长密钥”的设计，通常仍受原始口令空间限制。
- 复用要点：枚举时必须精确复刻补零、MD5 迭代、整数取模、左侧零填充和分块顺序；合法填充只能作为筛选条件，最终还应检查已知明文格式。
