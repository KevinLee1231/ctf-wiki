# Bomb 2: Mix Up

## 题目简述

题目给出多行长字符串，每行长度约 1000。每行绝大多数字符是噪声，出现次数最多的字符才是该行承载的明文；按行求众数即可恢复 flag。

## 解题过程

逐行统计字符频次，忽略行尾换行符，选择计数最大的字符：

```python
from collections import Counter

out = []
with open("input.txt", encoding="utf-8") as handle:
    for line in handle:
        line = line.rstrip("\r\n")
        if not line:
            continue
        char, count = Counter(line).most_common(1)[0]
        out.append(char)

print("".join(out))
```

如果某行出现并列，应回查题目生成逻辑或使用全局 flag 格式验证；本题数据中每行众数唯一。拼接结果为：

```text
UMDCTF-{w0w_y0u_4re_4bl4_t0_g3t_m0d3}
```

## 方法总结

本题的决定性操作是频率统计，归入通用编程而非密码学。处理行数据时要明确是否保留空格、制表符和 `\r`；只移除换行，避免把可能的有效字符一并 `strip()` 掉。
