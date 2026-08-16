# Beginner hacker 2

## 题目简述

本题先用固定 seed 打乱 flag 花括号内的字符，再使用长度为 12 的循环 XOR key 加密。flag 总长度是 12 的倍数。已知前缀 `shellmates{` 提供 11 个 key 字节，末尾 `}` 恰好对应第 12 个 key 字节，因此整个 XOR key 都可直接恢复；随后只需复现并逆转 Python 的 `random.shuffle`。

## 解题过程

### 恢复完整 XOR key

```python
encrypted = b'S64\xd1\xeeG\x95\xcf\rl\xe3n\x7f\x013\xd3\xf4M\x80\xff<@\xcd\x18h\te\xe2\xa3d\xad\x9f\r/\xf6X\x7f\x0cb\xcf\xe3O\x8d\xd2\x00J\xd9\x18W\x15b\xe2\xc0d\xbd\xde7V\xebn\x7f\x1d\x1f\xe2\xc6G\xc4\xe4-s\xc7V'
prefix = b"shellmates{"

key = bytearray(12)
for i, value in enumerate(prefix):
    key[i] = encrypted[i] ^ value

# len(encrypted) % 12 == 0，所以最后一字节使用 key[11]
key[11] = encrypted[-1] ^ ord("}")

shuffled_flag = bytes(
    value ^ key[i % 12]
    for i, value in enumerate(encrypted)
)
```

### 逆转确定性的 shuffle

源码用 `bytes_to_long(key)` 作为随机数种子。相同 Python 随机算法、相同 seed 和相同列表长度会产生同一排列。对下标列表执行一次 shuffle，即可得到“打乱后位置对应原位置”的映射，再把字符放回去：

```python
import random
from Crypto.Util.number import bytes_to_long

middle = shuffled_flag[len(prefix):-1]
permutation = list(range(len(middle)))
random.seed(bytes_to_long(bytes(key)))
random.shuffle(permutation)

original = bytearray(len(middle))
for shuffled_pos, original_pos in enumerate(permutation):
    original[original_pos] = middle[shuffled_pos]

flag = prefix + bytes(original) + b"}"
print(flag)
```

输出为：

```text
shellmates{ev3n_D0Uble_EnCRYti0N_Is_B4D_WHeN_wE_ar3_UsINg_ThE_$Am3_K3y!}
```

## 方法总结

- 核心技巧：已知格式完全恢复循环 XOR key，然后用相同 seed 重建排列并求逆。
- 识别信号：加密前的“随机打乱”若使用可恢复 key 派生 seed，就不是独立安全层。
- 复用要点：`shuffle` 原地改变列表；逆变换时应把第 `shuffled_pos` 个字符写回 `permutation[shuffled_pos]`，不能简单再次 shuffle 字符串。
