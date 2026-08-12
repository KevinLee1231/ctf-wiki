# DownUnderCTF 2022 ezpz-rev Writeup

## 题目简述

程序要求输入一个 $14\times14$ 的 `0/1` 棋盘。只有当每行、每列、每个不规则区域都恰有三个 `1`，并且任意两个 `1` 在八邻域内都不相邻时，程序才会读取 `flag-rev.txt`。题目核心是从校验函数恢复约束并求出一组合法棋盘。

## 解题过程

源码中的 `BOARD_DATA` 是游程编码，每两个字符表示“重复次数 + 区域字母”，例如 `5a4b` 展开为五个 `a` 和四个 `b`。完整字符串展开后有 196 个字符，区域编号为 `a` 到 `n`。

四个检查函数分别给出如下约束：

- 每一行的 14 个变量之和为 3；
- 每一列的 14 个变量之和为 3；
- 属于同一字母区域的变量之和为 3；
- 水平、垂直和对角方向相邻的两个变量不能同时为 1。

这些都是布尔约束，可直接交给 Z3。下面省略的 `BOARD_DATA` 使用题目源码中的原字符串：

```python
from z3 import Bool, If, Not, Or, Solver, Sum, is_true

N = 14
board_data = '5a4b3c2d5a4b4c1d2a1e3a3b4c1d2a1e1f2a3b3c1g1d3e5f1b2c2g1d1f2e4f6g1d7f3h3g1d4f6h3g1d2f1i4j1h2k2l1m1d3i2j5k2l2m2i3j4k3l2m2i3j4k3l2m1i5j2k2n2l2m1i4j5n4l'

areas = ''.join(int(board_data[i]) * board_data[i + 1]
                for i in range(0, len(board_data), 2))
x = [[Bool(f'x_{r}_{c}') for c in range(N)] for r in range(N)]
s = Solver()

for r in range(N):
    s.add(Sum([If(x[r][c], 1, 0) for c in range(N)]) == 3)
for c in range(N):
    s.add(Sum([If(x[r][c], 1, 0) for r in range(N)]) == 3)
for area in 'abcdefghijklmn':
    cells = [If(x[i // N][i % N], 1, 0)
             for i, value in enumerate(areas) if value == area]
    s.add(Sum(cells) == 3)

for r in range(N):
    for c in range(N):
        for dr, dc in ((0, 1), (1, -1), (1, 0), (1, 1)):
            nr, nc = r + dr, c + dc
            if 0 <= nr < N and 0 <= nc < N:
                s.add(Or(Not(x[r][c]), Not(x[nr][nc])))

assert s.check().r == 1
m = s.model()
rows = [''.join('1' if is_true(m.evaluate(x[r][c])) else '0'
                for c in range(N)) for r in range(N)]
print('\n'.join(rows))
print(''.join(rows))
```

一组合法答案为：

```text
01010000000010
00000101010000
10100000000100
00000001010001
10100000000100
00001010100000
01000000001001
00001010100000
00100000001010
10001010000000
00000000101010
01010100000000
00000000010101
00010101000000
```

将 14 行无分隔拼接后提交，程序输出：

```text
DUCTF{gr1d_puzzl3s_ar3_t00_ez_r1ght?}
```

## 方法总结

这是一道由逆向得到约束、再用 SMT 求解的网格题。要点是先正确展开区域游程编码，再把四类检查逐项翻译；八邻域约束只枚举右、左下、下、右下四个方向即可避免重复。该二进制还与 `ezpz-pwn` 共用，但本题只需满足前 196 字节的逻辑校验，不需要利用其中的 `gets` 溢出。
