# 变量迷城

## 题目简述

题目附件是一个 Java JAR。程序从环境变量读取整数 `x`、`y`，从 Java 系统属性读取 `brand`；只有三者满足校验条件后，程序才会用它们解密内置密文。

## 解题过程

### 还原校验条件

反编译 `Main` 类，可以看到 `brand` 必须等于 `0xGame`，而 `x`、`y` 需要满足：

$$
x^2+2y^2+3x+4y=7384462351178
$$

$$
5x^2+6y^2+7x+8y=22179606057658
$$

使用 SymPy 求解，并只保留整数解，避免把无关的复数根全部输出：

```python
from sympy import Eq, solve, symbols

x, y = symbols("x y")
equations = (
    Eq(x**2 + 2*y**2 + 3*x + 4*y, 7384462351178),
    Eq(5*x**2 + 6*y**2 + 7*x + 8*y, 22179606057658),
)

for sx, sy in solve(equations, (x, y)):
    if sx.is_integer is True and sy.is_integer is True:
        print(sx, sy)
```

输出为：

```text
114514 1919810
```

### 设置运行参数并解密

源码还表明，解密密钥由 `x.toString() + brand` 拼接而成，随后循环异或内置的 `encryptedFlag`。在 PowerShell 中设置环境变量，并通过 `-D` 传入 Java 系统属性：

```powershell
$env:x = "114514"
$env:y = "1919810"
java -Dbrand=0xGame -jar ".\变量迷城.jar"
```

得到：

```text
0xGame{9357863c-bcd8-ed3e-008f-2040d2e45043}
```

## 方法总结

本题的关键是区分环境变量与 Java 系统属性：`x`、`y` 由运行环境提供，`brand` 通过 `-Dbrand=...` 传入。先从反编译代码提取方程组并筛出整数解，再按源码要求设置三个参数，即可触发重复异或解密并得到 flag。
