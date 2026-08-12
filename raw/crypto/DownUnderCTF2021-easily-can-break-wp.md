# DownUnderCTF 2021 - Easily Can Break

## 题目简述

服务提供 AES-128-ECB 加密 oracle。每次把 32 字节 flag、用户输入和 16 字节 AES 密钥依次拼接，再用字符 `0` 补齐到 16 字节倍数后加密：

```python
def enc(user_input):
    cipher = AES.new(key, AES.MODE_ECB)
    plaintext = flag + user_input + key
    return b64encode(cipher.encrypt(pad(plaintext)))
```

密钥同时出现在明文尾部。ECB 对相同明文块产生相同密文块，因此可以通过选择输入，把“未知密钥前缀”与“已知前缀加一个猜测字节”对齐到同一个块，逐字节恢复 AES 密钥。

## 解题过程

flag 恰好占两个 AES 块。输入 15 个 `A` 时，第 3 个明文块为：

```text
AAAAAAAAAAAAAAA || key[0]
```

保存该请求的第 3 个密文块作为目标。然后依次发送：

```text
AAAAAAAAAAAAAAA || candidate
```

由于前两个 flag 块不变，当第 3 个密文块与目标相同时，`candidate` 就是 `key[0]`。恢复一个字节后把填充 `A` 减少一个，将已恢复前缀放入输入，再猜下一个字节。源码注释说明密钥是普通密码，因此官方 solver 只枚举可打印 ASCII。

```python
from base64 import b64decode

BLOCK = 16

def third_block(encoded):
    raw = b64decode(encoded)
    return raw[2 * BLOCK:3 * BLOCK]

known = b""
for remaining in range(16, 0, -1):
    prefix = b"A" * (remaining - 1)
    target = third_block(query(prefix))

    for value in range(33, 127):
        candidate = prefix + known + bytes([value])
        if third_block(query(candidate)) == target:
            known += bytes([value])
            break

key = known
```

取得 16 字节密钥后，对空输入请求一次。该响应的前两个密文块正好对应 32 字节 flag，直接 ECB 解密：

```python
from Crypto.Cipher import AES

ciphertext = b64decode(query(b""))
plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
flag = plaintext[:32]
print(flag.decode())
```

仓库中官方旧说明的最终字符串有前缀和末尾字符笔误；以挑战实际 `flag.txt` 为准：

```text
DUCTF{ECB_M0DE_K3YP4D_D474_L34k}
```

## 方法总结

本题是 ECB 字节对齐攻击的变体：未知内容不是通常的 secret suffix，而是加密密钥本身。识别信号是“攻击者可控输入 + 固定未知数据 + ECB + 未知数据再次进入明文”。只要能把目标字节推到块末尾，就可用密文块相等性作为确定性 oracle。根本修复是使用带随机 nonce 的认证加密，并绝不把密钥拼入待加密明文。
