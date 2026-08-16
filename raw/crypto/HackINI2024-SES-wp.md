# SES - Spooder Encryption System

## 题目简述

自定义加密把明文按 16 字节分块，并把前一个密文块直接当作下一块的 AES-ECB key。首块则使用最后一个明文块作为 key，形成环形依赖。明文末尾固定追加已知字符串 `Free Palestine.`，因此可以枚举 PKCS#7 padding 长度，恢复最后一个明文块并解开整个链。

## 解题过程

设明文块为 $P_0,\ldots,P_{n-1}$，密文块为 $C_0,\ldots,C_{n-1}$。源码实际执行：

$$
C_0=E_{P_{n-1}}(P_0)
$$

$$
C_i=E_{C_{i-1}}(P_i),\qquad i\ge1
$$

### 枚举最后一个明文块

未填充明文以 15 字节固定文本 `Free Palestine.` 结尾。若 padding 长度为 $i$，最后一块就是该固定文本末尾的 $16-i$ 字节，再接 $i$ 个值均为 `i` 的字节：

```python
known_suffix = b"Free Palestine."

for pad_len in range(1, 17):
    last_plain = known_suffix[pad_len - 1:] + bytes([pad_len]) * pad_len
```

用候选 `last_plain` 作为 AES key 解密 `C0`，出现 `shellmates{` 的候选即为正确末块。

### 逆向恢复所有明文块

对 $i\ge1$，前一个密文块已知，所以：

$$
P_i=D_{C_{i-1}}(C_i)
$$

得到最后一块后，首块也可由 $P_0=D_{P_{n-1}}(C_0)$ 恢复：

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

raw = open("enc", "rb").read()
ct = [raw[i:i + 16] for i in range(0, len(raw), 16)]
known_suffix = b"Free Palestine."

def decrypt_block(key, block):
    return AES.new(key, AES.MODE_ECB).decrypt(block)

last_plain = None
for pad_len in range(1, 17):
    candidate = known_suffix[pad_len - 1:] + bytes([pad_len]) * pad_len
    if decrypt_block(candidate, ct[0]).startswith(b"shellmates{"):
        last_plain = candidate
        break

assert last_plain is not None
pt = [b""] * len(ct)
pt[-1] = last_plain
for i in range(len(ct) - 1, 0, -1):
    pt[i] = decrypt_block(ct[i - 1], ct[i])
pt[0] = decrypt_block(pt[-1], ct[0])

message = unpad(b"".join(pt), 16)
print(message)
```

附件中的实际消息给出 flag：

```text
shellmates{N3v3r_stop_t4lK1nG_aB0Ut_P4l3$tinE}
```

`solution/README.md` 仍保留模板中的假 flag，以上结果由 `solution/flag`、加密源码和官方 solver 共同确认。

## 方法总结

- 核心技巧：利用已知消息后缀枚举最后一个明文块，再沿密文作 key 的链条逐块逆向解密。
- 识别信号：自定义分组模式把明文或密文直接当 AES key，且存在已知块或可枚举 padding 时，通常会形成可逆链。
- 复用要点：PKCS#7 padding 字节值等于 padding 长度；环形首块必须最后用恢复出的 $P_{n-1}$ 解密，不能按普通 CBC 处理。
