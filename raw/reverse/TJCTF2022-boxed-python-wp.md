# TJCTF2022 boxed-python

## 题目简述

主程序把输入转换成 `float64` 数组，调用编译后的 Python 扩展 `box.size()` 和 `box.check()`。生成源码表明，Numba 扩展内部保存随机整数矩阵 $A$ 与向量 $b=A x$，其中 $x$ 是 flag 的字节向量；只要恢复编译模块中的常量，就能把校验还原为线性方程组。

## 解题过程

`size()` 返回 26，而矩阵只有 $26-7=19$ 行，因此从扩展的只读数据中提取出 $19\times 26$ 个 `double` 作为 $A$，并提取 19 个 `double` 作为 $b$。原校验条件就是：

$$A x=b$$

19 条方程不足以唯一确定 26 个未知数，但 flag 格式提供了 7 个已知字符：前六字节为 `tjctf{`，最后一字节为 `}`。将这七个约束补成单位行后，方程组变为 $26\times 26$：

```python
A = np.concatenate((A, np.zeros((7, 26))))
for row, col in enumerate([0, 1, 2, 3, 4, 5, 25], start=19):
    A[row, col] = 1
b = np.concatenate((b, np.array(list(b'tjctf{}'))))

x = np.linalg.solve(A, b).round().astype(np.uint8)
print(bytes(x).decode())
```

浮点解四舍五入回整数字节后得到 `tjctf{1ts_ju5t_l1n3s_lm40}`，代回原矩阵也满足比较。

## 方法总结

编译成 `.so` 并不会隐藏数值模型所必需的矩阵常量。分析此类扩展时，先从 Python 包装层确定输入类型、维度和数学关系，再在二进制中定位连续常量，往往比逐条反编译数值代码更直接。欠定系统也不等于不可解：CTF flag 的固定前后缀正好可以补足缺失约束。
