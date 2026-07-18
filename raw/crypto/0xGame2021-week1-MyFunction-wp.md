# week1MyFunction

## 题目简述

压缩包中的 `output.txt` 每行给出一个浮点数 $y=x\ln x$，其中 $x$ 是一个 ASCII 字符码。需要逐行恢复 $x$ 并拼接明文。

## 解题过程

对 $f(x)=x\ln x$ 求导：

$$
f'(x)=\ln x+1.
$$

在题目使用的整数范围 $1\le x<128$ 内，$f(x)$ 严格递增，因此每个输出只对应一个 ASCII 码。与其人为设置容易受浮点误差影响的阈值，可以枚举全部候选并选择误差最小者：

```python
from math import log
from pathlib import Path

chars = []
for line in Path("output.txt").read_text().splitlines():
    y = float(line)
    x = min(range(1, 128), key=lambda n: abs(n * log(n) - y))
    chars.append(chr(x))

print("".join(chars))
```

输出：

```text
0xGame{YouH4veKn0wedPy7honL081s<y=ln(x)>InM47hs}
```

仓库自带 `solution.py` 末尾的注释误写成了另一题的 flag；以上结果由 `MyFunction.zip` 中的实际 `output.txt` 逐行恢复，不能采用那条旧注释。

## 方法总结

离散字符域上的单调函数不必求解析反函数，有限枚举即可精确恢复。处理浮点输出时，选择全域最小误差通常比固定容差更稳健，同时要用实际数据而不是源码中的过期注释核验结果。
