# godness_dance

## 题目简述

程序读取 28 个小写字母，先检查字符频次，再构造输入字符串的后缀数组并与常量 `ta` 比较。频次表要求 `a` 到 `z` 各出现一次，只有 `i` 和 `u` 各出现两次；目标后缀数组为：

```text
2, 26, 17, 28, 24, 11, 21, 10, 16, 20, 19, 18, 3, 8,
6, 12, 9, 14, 13, 22, 4, 27, 15, 23, 1, 25, 7, 5
```

这里不需要逆向复杂的倍增后缀数组实现。后缀数组已经给出了各后缀按字典序排列后的起始位置，而字符多重集合也完全已知，可以直接按排名把首字符放回原位置。

## 解题过程

后缀数组 `sa[r] = p` 表示从位置 $p$ 开始的后缀是第 $r$ 小的后缀。字典序首先比较后缀的首字符，所以按排名观察 `s[sa[r]]`，它必然是非递减序列。题目又固定了全部字符计数，因此这个序列恰好就是已知字符多重集合的排序结果。

将排序后的 28 个字符依次赋给 `ta` 指定的位置：

```python
target_sa = [
    2, 26, 17, 28, 24, 11, 21, 10, 16, 20, 19, 18, 3, 8,
    6, 12, 9, 14, 13, 22, 4, 27, 15, 23, 1, 25, 7, 5,
]

counts = {chr(ord("a") + i): 1 for i in range(26)}
counts["i"] = 2
counts["u"] = 2
ordered = "".join(ch * counts[ch] for ch in sorted(counts))

answer = [""] * 28
for ch, position in zip(ordered, target_sa):
    answer[position - 1] = ch
candidate = "".join(answer)

check_sa = sorted(range(1, 29), key=lambda p: candidate[p - 1:])
assert check_sa == target_sa
print(candidate)
```

得到输入：

```text
waltznymphforquickjigsvexbud
```

重新计算完整后缀数组而不只检查首字符，结果与 `ta` 完全一致；同时 `i`、`u` 各出现两次，其余字母各一次。程序最终输出：

```text
SCTF{waltznymphforquickjigsvexbud}
```

## 方法总结

目标后缀数组与已知字符多重集合组合起来，已经泄露了每个位置的字符：第 $r$ 小后缀的首字符就是第 $r$ 个排序字符。直接赋值后仍要用真实的后缀排序重新验证，因为首字符相同的 `i`、`u` 需要由后续字符决定相对顺序。本题展示了一个常见逆向化简：不要复刻检查器的全部算法，先判断输出约束是否已经能直接反推出输入。
