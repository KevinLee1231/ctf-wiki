# Break the Chain

## 题目简述

程序要求输入 46 字节密码，并把每个输入字节放入链式状态中。校验不是独立逐字节比较：当前字节还与前一个字节共同参与异或和按位或，因此题名中的 “Chain” 指向这种前后依赖。

## 解题过程

反编译校验循环可归纳为：首字节的前态为 `0x42`，之后前态等于上一输入字节。令当前字节为 $x_i$、前态为 $p_i$，程序计算：

$$
t_i=(x_i+p_i)\bmod256
$$

并比较：

$$
x_i\oplus t_i,\qquad x_i\lor t_i
$$

两组期望数组位于文件偏移 `0x2080` 和 `0x20c0`，存储时又统一异或了 `0x42`。由于每一位只依赖前一个已恢复字符，可以从左到右枚举可打印 ASCII：

```python
from pathlib import Path

binary = Path("break_the_chain").read_bytes()
expected_xor = binary[0x2080:0x2080 + 46]
expected_or = binary[0x20C0:0x20C0 + 46]

paths = [b""]
for left, right in zip(expected_xor, expected_or):
    next_paths = []
    for path in paths:
        previous = path[-1] if path else 0x42
        for current in range(0x20, 0x7F):
            total = (current + previous) & 0xFF
            if (
                (current ^ total) == (left ^ 0x42)
                and (current | total) == (right ^ 0x42)
            ):
                next_paths.append(path + bytes([current]))
    paths = next_paths

assert len(paths) == 1
print(paths[0].decode())
```

唯一可打印路径为：

```text
Lc1614Q4GZb[LoGiC_IsnT_B0ring!]c11H72n7A3514F8
```

最终 flag：

```text
UMDCTF-{Lc1614Q4GZb[LoGiC_IsnT_B0ring!]c11H72n7A3514F8}
```

其 SHA-256 与 README 摘要一致。

## 方法总结

链式校验仍然可以逐步求解，只要状态依赖是单向的。与其一次性暴力搜索 $95^{46}$ 个字符串，不如把“前一字符”作为已知状态，每轮只枚举 95 个可打印字符，并同时用两条位运算约束剪枝。
