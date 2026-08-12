# DownUnderCTF 2020 - Cosmic Rays

## 题目简述

附件实现一种类似 IGE 的自定义 AES 模式。flag 去掉外层后正好 32 字节，被拆成两个初始状态；已知明文还包含实际 AES key 的十六进制表示。输出文件中的 key 缺失 3 个十六进制半字节，大部分密文也被破坏，但首个密文块完整、末块仅缺 2 个半字节。目标是利用已知明文和分组递推关系补回 key、密文与两个初始状态。

## 解题过程

令已知明文分组为 $P_i$，输出密文分组为 $Y_i$，AES 内部状态为 $U_i$。题目代码等价于：

$$
U_i=E_K(P_i\oplus U_{i-1}),\qquad Y_i=P_{i-1}\oplus U_i.
$$

其中 $U_0$ 是 flag 前 16 字节，$P_0$ 是 flag 后 16 字节。

### 恢复 AES key

key 缺 3 个半字节，末尾密文块缺 2 个半字节，总搜索量是 $16^5$。对每个候选 key 与末块 $Y_t$，利用末尾三个已知明文分组计算前一密文块：

$$
Y_{t-1}=D_K(Y_t\oplus P_{t-1})\oplus P_t\oplus P_{t-2}.
$$

虽然 $Y_{t-1}$ 也有缺失，但仍保留了足够多的半字节；只比较未损坏位置，就能以极低误报率筛出唯一 key。

```python
from itertools import product
from Crypto.Cipher import AES

HEX = "0123456789abcdef"

def fill_missing(text, replacements):
    for value in replacements:
        text = text.replace("▒", value, 1)
    return text

def matches_partial(pattern, candidate):
    return all(expected == "▒" or expected == actual
               for expected, actual in zip(pattern, candidate))

def aes_decrypt(block, key):
    return AES.new(key, AES.MODE_ECB).decrypt(block)

def recover_key(corrupted_key, corrupted_ct, plaintext_for_key):
    for guess in product(HEX, repeat=5):
        key = bytes.fromhex(fill_missing(corrupted_key, guess[:3]))
        plaintext = plaintext_for_key(key)
        last = bytes.fromhex(fill_missing(corrupted_ct[-32:], guess[3:]))
        previous = bytes(a ^ b ^ c for a, b, c in zip(
            aes_decrypt(bytes(x ^ y for x, y in zip(last, plaintext[-32:-16])), key),
            plaintext[-16:],
            plaintext[-48:-32],
        ))
        if matches_partial(corrupted_ct[-64:-32], previous.hex()):
            return key, last
    raise ValueError("key not found")
```

### 向前恢复全部密文与初始状态

得到 key 和最后一个完整密文块后，反复应用同一递推式即可从后向前补齐所有 $Y_i$。首两个密文块恢复后：

$$
U_1=D_K(Y_2\oplus P_1)\oplus P_2,
$$

$$
U_0=D_K(U_1)\oplus P_1,\qquad P_0=U_1\oplus Y_1.
$$

按照源码传参顺序，flag 内部内容为 $U_0\|P_0$，最终得到：

```text
DUCTF{IVs_4r3nt_s3cret!_but_fl4gs_ar3!}
```

## 方法总结

面对损坏的密码输出，不应只统计缺失量，还要寻找能从完整边界块向内递推的结构。本题利用已知明文把 5 个缺失半字节降为可行枚举，再用残存半字节作为候选 key 的校验 oracle；恢复 key 后，自定义模式的可逆分组关系足以重建全部密文和隐藏初始状态。
