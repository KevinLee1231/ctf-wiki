# Time Capsule

## 题目简述

题目把 flag 连续做 42 轮列换位，再使用当前时间作为 Python `random` 的种子生成异或流。作者把用于播种的 18 字节时间戳附在密文末尾，并仅与常量 `0x42` 异或，因此第二层的“随机种子”可以直接恢复。

第一层密钥虽然由 8 个不同随机字节组成，但算法只使用这些字节的相对大小来决定 8 列顺序，实际密钥空间只有 $8!$。

## 解题过程

第二层返回：

```python
now = str(time.time()).encode()
now += b"0" * (18 - len(now))
random.seed(now)
key = [random.randrange(256) for _ in message]

result = [
    m ^ k
    for m, k in zip(
        message + now,
        key + [0x42] * len(now),
    )
]
```

所以密文最后 18 字节满足：

$$
c_i = \text{now}_i \oplus 0x42.
$$

先恢复 `now`，再用完全相同的字节串初始化 Python PRNG，即可重建前半段密钥流：

```python
encrypted = Path("flag.enc").read_bytes()
now = bytes(value ^ 0x42 for value in encrypted[-18:])
body = encrypted[:-18]

random.seed(now)
stream = bytes(random.randrange(256) for _ in body)
stage_one = bytes(a ^ b for a, b in zip(body, stream)).decode()
```

第一层先按 8 个密钥字节排序，再依次输出相应位置的字符：

```python
order = sorted(zip(key, range(8)))
for _, column in order:
    for index in range(column, len(message), 8):
        result += message[index]
```

密钥的具体数值不影响结果，只有排序后的列编号有意义。枚举 `permutations(range(8))`，根据消息长度计算每一列在密文中占多少字符，将列块放回原下标，就能逆转一轮：

```python
def decrypt_round(ciphertext, order):
    groups = []
    offset = 0
    remainder = len(ciphertext) % 8

    for column in order:
        size = len(ciphertext) // 8
        if column < remainder:
            size += 1
        groups.append(ciphertext[offset:offset + size])
        offset += size

    plain = [""] * len(ciphertext)
    for group, column in zip(groups, order):
        for pos, value in zip(range(column, len(plain), 8), group):
            plain[pos] = value
    return "".join(plain)
```

对每个候选顺序逆转 42 次，用 `SEKAI{` 前缀和 `}` 结尾筛选：

```python
for order in itertools.permutations(range(8)):
    candidate = stage_one
    for _ in range(42):
        candidate = decrypt_round(candidate, order)
    if candidate.startswith("SEKAI{") and candidate.endswith("}"):
        print(candidate)
```

得到：

```text
SEKAI{T1m3_15_pr3C10u5_s0_Enj0y_ur_L1F5!!!}
```

## 方法总结

公开种子生成的伪随机流不提供保密性，更不能把种子随密文以固定异或形式附带。分析置换密码时还应区分“原始密钥取值”与“真正影响输出的等价类”：本题 8 个随机字节只诱导一个排列，密钥空间从看似巨大的 $256^8$ 降为 $8!$，可以直接穷举。
