# leannum

## 题目简述

附件是 Lean 4 编译出的 ELF、生成 C 文件以及 `.olean/.ilean` 中间产物。程序读取一行 81 个字符，转换成 $9\times9$ 数组，再调用 `check()`。Lean 运行时对象和生成 C 代码比较冗长，但保留的符号名如 `l_fromString`、`l_check`、`l_target` 足以恢复逻辑。

题目不是标准数独：除了每行、每列互异，还要求 9 条环绕对角线互异，没有 $3\times3$ 宫约束。

## 解题过程

### 恢复输入编码

生成 C 中的 `l_fromString` 对每个字符执行：

```c
value = char_code - 48;
```

随后要求每个值严格小于 9，且总长度恰好为 81。因此合法输入字符是 `0` 到 `8`，而不是常规数独的 `1` 到 `9`。

### 恢复检查条件

结合 `l_check` 的符号化辅助函数、数组下标和 `.ilean` 引用信息，可整理为：

```python
for i in range(9):
    Distinct(X[i])                           # 行
    Distinct([X[j][i] for j in range(9)])   # 列
    Distinct([X[j][(i + j) % 9]
              for j in range(9)])           # 环绕对角线
```

最后还逐格检查 `l_target` 中所有非空提示。恢复成普通 `1..9` 表示后，题目实例为：

```text
7 0 2 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0
0 0 0 0 6 0 0 0 0
5 0 6 0 3 0 0 0 0
0 0 0 0 0 0 1 3 0
0 0 0 0 0 0 8 0 6
0 4 0 0 0 0 5 0 0
0 0 8 5 0 2 0 0 0
0 5 0 0 0 0 0 0 0
```

这里的 0 表示没有提示，不是要填入棋盘的数字。

### 用 Z3 求解

建立 81 个取值 $1$ 到 $9$ 的整数变量：

```python
from z3 import And, Distinct, Int, Solver

X = [[Int(f"x_{r}_{c}") for c in range(9)]
     for r in range(9)]
s = Solver()

for r in range(9):
    for c in range(9):
        s.add(And(1 <= X[r][c], X[r][c] <= 9))

for i in range(9):
    s.add(Distinct(X[i]))
    s.add(Distinct([X[j][i] for j in range(9)]))
    s.add(Distinct([X[j][(i + j) % 9]
                    for j in range(9)]))

for r in range(9):
    for c in range(9):
        if instance[r][c] != 0:
            s.add(X[r][c] == instance[r][c])
```

求得：

```text
762819354
825976413
431765982
596438271
687294135
279153846
943681527
318542769
154327698
```

程序内部接受 `0..8`，所以每个数字都要减 1，再按行连接：

```text
651708243714865302320654871485327160576183024168042735832570416207431658043216587
```

程序输出 `Yes`，对应 flag 为：

```text
R3CTF{651708243714865302320654871485327160576183024168042735832570416207431658043216587}
```

Lean 生成物的动态定位和 Z3 原始约束可参考 [R3CTF leannum Writeup](https://www.z221x.website/article/page-10)。本文已根据仓库中的 `Main.c` 和 `.ilean` 补足字符减 48、长度/范围检查、环绕下标和最终减 1 编码。

## 方法总结

处理 Lean 4 编译产物时，不必从运行时引用计数代码逐指令硬啃。优先利用仍保留的函数符号、生成 C 和 `.ilean` 的源码引用关系，把高阶数组操作还原成约束。另一个常见陷阱是把题目误当标准数独：本题没有宫约束，却有 9 条模 9 环绕对角线。最后还要区分求解时便于表达的 `1..9` 与程序实际输入的 `0..8`。
