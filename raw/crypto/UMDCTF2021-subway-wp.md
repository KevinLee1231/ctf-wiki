# Subway

## 题目简述

题目给出一段由数字和字母混合组成的替换密文。字符集恰好有 36 个符号，提示使用由 `0-9a-z` 构成的环形字母表，而不是普通的 26 字母凯撒密码。

## 解题过程

建立：

```text
0123456789abcdefghijklmnopqrstuvwxyz
```

作为 36 符号字母表。枚举循环位移时，应让数字和字母共同参与同一模 $36$ 运算，而不是分别轮转。对密文中的每个字母数字字符按其索引减去候选偏移，标点原样保留：

```python
alphabet = "0123456789abcdefghijklmnopqrstuvwxyz"

for shift in range(36):
    out = []
    for ch in ciphertext.lower():
        if ch in alphabet:
            out.append(alphabet[(alphabet.index(ch) - shift) % 36])
        else:
            out.append(ch)
    print(shift, "".join(out))
```

唯一具有完整 flag 结构且语义通顺的结果是：

```text
UMDCTF-{not_a_simple_substitution_this_time}
```

## 方法总结

看到数字和字母同时参与替换时，应先统计有效字符集大小。本题是模 $36$ 的统一轮转；若把数字和字母拆成两个表，会破坏跨边界映射。枚举空间只有 36，直接以格式和可读性筛选即可。
