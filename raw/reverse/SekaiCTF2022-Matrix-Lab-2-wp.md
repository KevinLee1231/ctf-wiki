# Matrix Lab 2

## 题目简述

题目提供一个 Windows PE。程序会尝试启动 MATLAB Engine，没有安装 MATLAB 时只显示 `Unknown error`，但 PE 中大量 `Py_*` 符号表明它实际上是 PyInstaller 打包的 Python 程序。

恢复脚本后可以看到，flag 的 16 字节主体先逐字节异或 42、排成 $4\times4$ 矩阵，再经过转置、旋转和矩阵乘法。目标矩阵已硬编码，因此可以直接求逆。

## 解题过程

先用 `pyinstxtractor.py` 解包 `Matrix_Lab_2.exe`，再用与字节码版本匹配的 `uncompyle6` 反编译提取出的 `Matrix_Lab_2.pyc`。核心检查可整理为：

```python
A = [ord(i) ^ 42 for i in flag[6:-1]]
B = matlab.double([A[i:i + 4] for i in range(0, len(A), 4)])

C = magic(4) + pascal(4)
D = [
    [2094, 2962, 1014, 2102],
    [2172, 3955, 1174, 3266],
    [3186, 4188, 1462, 3936],
    [3583, 5995, 1859, 5150],
]

C * rot90(transpose(B), 1337) == D
```

MATLAB 的四阶 magic square 与 Pascal 矩阵相加后为：

```text
C = [
    [17,  3,  4, 14],
    [ 6, 13, 13, 12],
    [10, 10, 12, 22],
    [ 5, 18, 25, 21],
]
```

令 $M=\operatorname{rot90}(B^T,1337)$，则 $CM=D$，所以 $M=C^{-1}D$。`rot90` 的旋转次数按 4 取模，$1337\bmod4=1$，故：

$$
B=\left(\operatorname{rot90}(M,3)\right)^T
$$

可以直接用 NumPy 求解：

```python
import numpy as np

C = np.array([
    [17, 3, 4, 14],
    [6, 13, 13, 12],
    [10, 10, 12, 22],
    [5, 18, 25, 21],
], dtype=np.int64)

D = np.array([
    [2094, 2962, 1014, 2102],
    [2172, 3955, 1174, 3266],
    [3186, 4188, 1462, 3936],
    [3583, 5995, 1859, 5150],
], dtype=np.int64)

M = np.rint(np.linalg.solve(C, D)).astype(np.int64)
B = np.rot90(M, 3).T

middle = "".join(chr(int(value) ^ 42) for value in B.flat)
print(f"SEKAI{{{middle}}}")

# 正向复算，确认矩阵方向没有写反。
assert np.array_equal(C @ np.rot90(B.T, 1), D)
```

恢复出的矩阵为：

```text
[
    [103,  30, 29, 102],
    [ 30, 104, 27,  31],
    [ 30, 125, 25, 121],
    [ 26, 103, 25,  11],
]
```

逐元素异或 42 后得到：

```text
SEKAI{M47L4B154W3S0M3!}
```

反编译结果里还有一个控制流瑕疵：成功变量 `outcome` 在前一个 `if` 内被设置，但打印成功信息的分支写成了同一条件链里的 `elif outcome`，因此正常输入也不会按作者意图进入该分支。求解依赖的是固定矩阵关系，不需要让原程序真正打印成功。

## 方法总结

这题的第一层障碍是识别 PyInstaller，而不是配置完整 MATLAB 环境；第二层障碍才是矩阵变换。看到 `Py_*` 符号和嵌入式 Python 字节码后，应优先恢复源逻辑。

矩阵逆向最重要的是写清转置与旋转的先后顺序，并对最终结果做一次正向复算。浮点线性方程求解后取整是可行的，但必须用 $C\operatorname{rot90}(B^T,1)=D$ 验证，避免静默接受数值误差或错误方向。
