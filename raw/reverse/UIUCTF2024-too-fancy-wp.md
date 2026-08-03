# UIUCTF 2024 tooooo fancy 😏

## 题目简述

题目只提供 Tcl Pro 编译后的 `main.tbc` 和运行环境，程序要求输入一行 flag，再回答输入是否足够“fancy”。核心校验被 Tcl 字节码、动态变量名和大量 MD5 调用掩盖；还原后本质上是一个整数线性方程组。

发布程序中的目标向量有 36 项，权重有 $36^2=1296$ 项，候选 flag 长度也固定为 36。部分简略说明把它写成 37 阶矩阵，但实际附件应以 36 阶为准。

## 解题过程

先识别 `.tbc` 为 Tcl Pro 字节码。可使用 [corbamico/tbcload](https://github.com/corbamico/tbcload) 的 `decompile` 子命令把文件反汇编成人类可读的 Tcl 指令；该工具支持普通和详细模式，后者更便于恢复常量、循环和动态变量访问：

```bash
tbcload decompile --detail main.tbc
```

根据反汇编结果重建 Tcl 逻辑。程序先把输入逐字符转为码点数组：

```tcl
set input [list]
foreach char [split [gets stdin] ""] {
  lappend input [scan $char "%c"]
}
```

随后出现 1296 个 `0..255` 的整数。它没有把这些数直接放进二维数组，而是反复计算 MD5，把每个哈希字符串当作 Tcl 变量名：

```tcl
set set set
foreach wat $weights {
  set set [md5::md5 -hex $set]
  set $set $wat
}
```

因此 MD5 不承担密码学校验，只是在构造一条“哈希值作为下一个变量名”的链。后面的循环通过 Lazy Caterer 序列相关的起点公式和逐步哈希，从这条链中按特定顺序取回权重。

把访问顺序整理成矩阵后，权重沿反对角线方向填充。以 $4\times4$ 且权重依次为 `1..16` 为例，得到：

```text
 7  4  2  1
11  8  5  3
14 12  9  6
16 15 13 10
```

发布附件的填充过程可直接写成：

```python
n = 36
A = [[0] * n for _ in range(n)]
i, j = 0, n - 1

for value in weights:
    A[i][j] = value
    i += 1
    j += 1
    if j >= n and i < n:
        j = n - i - 1
        i = 0
    if i >= n:
        i = n - j + 1
        j = 0
```

内层 Tcl 循环对输入码点倒序访问，但权重的行访问也采用了对应的逆序。整理索引后，每轮得到的 `epic` 就是矩阵一行与输入向量的点积。最后的判断为：

```tcl
set unfanciness [expr $unfanciness | $epic ^ [lindex $wtf $n]]
```

只有所有行都满足 `epic == wtf[n]`，逐行异或再按位或的结果才会保持为零。于是问题等价于：

$$
A x=b,
$$

其中 $x$ 是 36 个输入字符的码点，`wtf` 就是目标向量 $b$。矩阵可逆，所以使用精确整数/有理数求解，避免浮点逆矩阵造成字符四舍五入错误：

```python
from sympy import Matrix

A = Matrix(matrix_rows)
b = Matrix(target)
x = A.LUsolve(b)

assert all(value.q == 1 for value in x)
assert A * x == b
flag = "".join(chr(int(value)) for value in x)
print(flag)
```

从 `main.tbc` 恢复出的 1296 个权重和 36 个目标值代入后，结果为：

```text
uiuctf{1_hope_that_tcls_y0ur_f4ncy!}
```

重新计算 $Ax$ 与 $b$ 完全相等，因此每一行的异或差都为零，程序输出成功提示。

## 方法总结

这道题的“fancy”主要来自两层表现形式：外层是 Tcl 字节码，内层是用 MD5 链生成动态变量名。先反汇编 `.tbc`，再把动态变量访问还原成确定的权重序列，就能看出循环只是在计算 36 阶矩阵乘法。

MD5 在这里不是需要碰撞或逆向的散列；它只负责索引。真正需要保持准确的是反对角线填充顺序、输入倒序索引和矩阵阶数。使用 SymPy 的精确求解并回代验证，比浮点计算 `A^{-1}b` 更稳妥。
