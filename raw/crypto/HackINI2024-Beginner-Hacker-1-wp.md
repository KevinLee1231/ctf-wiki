# Beginner hacker 1

## 题目简述

题目用长度为 13 的随机 key 对 flag 做循环异或，并给出密文字节。flag 长度恰好是 13 的倍数，同时已知格式为 `shellmates{...}`。循环异或本身没有隐藏 key 的对齐关系，已知前缀和末尾右花括号足以恢复 12 个 key 字节，只剩 1 字节需要枚举。

## 解题过程

### 利用已知明文恢复 key

循环异或满足：

$$
c_i=p_i\oplus k_{i\bmod 13}
$$

因此有 $k_{i\bmod13}=c_i\oplus p_i$。已知前缀 `shellmates{` 长 11 字节，可恢复 `key[0]` 至 `key[10]`。又因为密文长度是 13 的倍数，最后一个明文字节 `}` 对应 `key[12]`：

```python
encrypted = b'\x1c62\x8f|i\x9a\xadw\xba\x06N\x85\x08+d\xb0C[\xa9\xed|\x8dM\x12\xb4\\\r$\xbct4\x9e\xaaM\x872\x0b\x85.\x12\x00\xa2iW\xa4\xb4W\xfd3 \xa3\x00\x0b\x08\xb1O \x9e\x9ag\xbbN^\xa7'
known_prefix = b"shellmates{"

key = bytearray(13)
for i, value in enumerate(known_prefix):
    key[i] = encrypted[i] ^ value

key[12] = encrypted[-1] ^ ord("}")
```

### 枚举唯一未知字节

只剩 `key[11]` 未知，搜索空间仅为 256。由于前缀和末尾在每次枚举中本来就固定，仅检查 `startswith()` 与 `endswith()` 会错误地保留全部 256 项。应先用常见 flag 字符集筛出少量候选，再根据明文语义判断：

```python
import re

for guess in range(256):
    key[11] = guess
    plain = bytes(
        value ^ key[i % len(key)]
        for i, value in enumerate(encrypted)
    )
    if re.fullmatch(rb"shellmates\{[A-Za-z0-9_$!]+\}", plain):
        print(plain)
```

字符集约束留下 3 项，其中只有下面这一项构成连贯的 leetspeak 句子；另外两项在多个固定间隔位置出现同步错字：


```text
shellmates{1_gu3SS_R4nD0mn3Ss_d0es_NOt_ALWAyS_mE4N_yoU_R_$eCur3!}
```

## 方法总结

- 核心技巧：用循环 XOR 的已知明文直接还原对应 key 字节，再枚举极小的剩余空间。
- 识别信号：固定 flag 前缀、已知末尾字符、key 很短且重复使用，都会泄露 key 的确定字节。
- 复用要点：先计算每个已知字符对应的 key 下标；题目中“明文长度是 key 长度的倍数”正是确定末尾 `}` 对齐位置的关键信息。
