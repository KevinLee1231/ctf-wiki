# week1lowerBase64

## 题目简述

出题脚本先执行标准 Base64 编码，再对结果调用 `.lower()`。Base64 区分大小写，因此这一步会丢失每个字母原本是大写还是小写的信息；数字、连字符和填充符不受影响。源码同时限制 flag 只包含小写字母、数字、`-`、`{`、`}`，可用这一明文字符集约束恢复大小写。

```python
assert all(char in "abcdefghijklmnopqrstuvwxyz0123456789-{}" for char in flag)
print(b64encode(flag.encode()).decode().lower())
```

## 解题过程

Base64 每 4 个字符独立编码 3 个字节，所以可以按 4 字符分组，枚举组内每个字母的两种大小写。仅保留解码后所有字节都落在已知 flag 字符集中的候选，再组合各组并检查完整格式。

```python
import base64
import itertools
import re

cipher = "mhhnyw1lezviodq1ntkxltmwmditngjlny1hzgi5lwu4m2q1ntcymtblnx0="
allowed = set(b"abcdefghijklmnopqrstuvwxyz0123456789-{}")


def decode_block(block):
    choices = []
    for char in block:
        if char.isalpha():
            choices.append((char.lower(), char.upper()))
        else:
            choices.append((char,))

    candidates = set()
    for variant in itertools.product(*choices):
        encoded = "".join(variant)
        try:
            raw = base64.b64decode(encoded, validate=True)
        except ValueError:
            continue
        if all(byte in allowed for byte in raw):
            candidates.add(raw)
    return sorted(candidates)


blocks = [cipher[i:i + 4] for i in range(0, len(cipher), 4)]
candidate_groups = [decode_block(block) for block in blocks]

for parts in itertools.product(*candidate_groups):
    flag = b"".join(parts)
    if re.fullmatch(rb"0xgame\{[a-z0-9-]+\}", flag):
        print(flag.decode())
```

输出为：

```text
0xgame{5b845591-3002-4be7-adb9-e83d557210e5}
```

恢复出的标准 Base64 字符串为 `MHhnYW1lezViODQ1NTkxLTMwMDItNGJlNy1hZGI5LWU4M2Q1NTcyMTBlNX0=`，可再次解码验证结果。

## 方法总结

- 核心技巧：按 Base64 四字符分组枚举丢失的大小写位，再用已知明文字符集剪枝。
- 识别信号：字符串完全小写但形态像 Base64，且直接解码得到乱码或不符合 flag 格式。
- 复用要点：不要对整串做 $2^n$ 级全局爆破；分组恢复后仍要用前缀、后缀和字符集做全局一致性校验。
