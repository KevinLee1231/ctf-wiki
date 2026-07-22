# ezlegendre

## 题目简述

程序把每个明文 bit $b\in\{0,1\}$ 编码为 $(a+b)^e\bmod p$，其中每次使用一个随机奇素数 $e$。参数特意满足 $a$ 是模 $p$ 二次非剩余，而 $a+1$ 是二次剩余，所以 ciphertext 的勒让德符号直接泄漏原 bit。

## 解题过程

对非零 $x$，欧拉判别法给出：

$$
x^{(p-1)/2}\equiv
\begin{cases}
1\pmod p, & x\text{ 是二次剩余},\\
-1\pmod p, & x\text{ 是二次非剩余}.
\end{cases}
$$

又因为 $e$ 为奇数：

$$
\left(\frac{x^e}{p}\right)=\left(\frac{x}{p}\right)^e
=\left(\frac{x}{p}\right).
$$

因此结果为 1 时原 bit 是 1，结果为 $p-1$ 时原 bit 是 0。附件 `task.py` 已把 `p`、`a` 和完整 ciphertext 写在末尾的三引号输出中；无需在 WP 里复制数百个大整数，可直接解析：

```python
import ast
import re
from pathlib import Path

source = Path("task.py").read_text(encoding="utf-8")
transcript = source.split("'''", 2)[1]

p = int(re.search(r"^p = (\d+)$", transcript, re.MULTILINE).group(1))
a = int(re.search(r"^a = (\d+)$", transcript, re.MULTILINE).group(1))
ciphertext = ast.literal_eval(
    next(line for line in transcript.splitlines() if line.startswith("["))
)

assert pow(a, (p - 1) // 2, p) == p - 1
assert pow(a + 1, (p - 1) // 2, p) == 1

bits = "".join(
    "1" if pow(value, (p - 1) // 2, p) == 1 else "0"
    for value in ciphertext
)
flag = bytes(
    int(bits[offset:offset + 8], 2)
    for offset in range(0, len(bits), 8)
)
print(flag)
```

输出：

```text
moectf{minus_one_1s_n0t_qu4dr4tic_r4sidu4_when_p_mod_f0ur_equ41_to_thr33}
```

## 方法总结

加密虽然每 bit 使用不同随机指数，但奇数次幂不会改变勒让德符号。只要两个候选底数分别落在剩余与非剩余集合中，欧拉判别就成为无密钥判 bit 的 oracle；随机指数没有隐藏这一分类信息。
