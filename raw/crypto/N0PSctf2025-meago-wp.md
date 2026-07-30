# Meago

## 题目简述

服务把 flag 转成大整数 $m$，再按十进制位数缩放成 $0<x_0<1$ 的高精度实数。它公开随机的 $y_0$，并允许反复调用一个非线性迭代；前 5 次隐藏结果，从第 6 次开始输出新的 $y$。目标是从迭代的收敛值反推出 $x_0$，再恢复 flag。

## 解题过程

源码中的一次 `meago` 调用不是同时赋值，而是先更新 $y$，再用新 $y$ 更新 $x$：

$$
y'=(xy^2)^{1/3}
$$

$$
x'=(x{y'}^2)^{1/3}
$$

在对数域中令 $u=\log x$、$v=\log y$，则迭代变为线性关系：

$$
v'=\frac{u+2v}{3},\qquad
u'=\frac{5u+4v}{9}
$$

由此可直接验证：

$$
3u'+4v'=3u+4v
$$

也就是说，$3\log x+4\log y$ 在迭代中保持不变。反复调用后 $x$ 与 $y$ 收敛到同一个值 $l$，因此：

$$
7\log l=3\log x_0+4\log y_0
$$

进一步得到：

$$
l=(x_0^3y_0^4)^{1/7}
$$

连接服务后保存最初公开的 $y_0$，调用数百次 `M` 使输出稳定为 $l$。于是 $x_0$ 是方程

$$
(x^3y_0^4)^{1/7}-l=0
$$

的正根。虽然可以形式化写成 $x_0=(l^7/y_0^4)^{1/3}$，但直接做有限精度幂运算容易在十进制尾部产生误差。官方求解脚本使用高精度实数和 Halley 迭代：

```sage
from Crypto.Util.number import long_to_bytes


def halley(function, initial, digits):
    derivative(x) = diff(function(x), x)
    second_derivative(x) = diff(function(x), x, 2)
    current = initial
    for _ in range(20):
        numerator = 2 * function(current) * derivative(current)
        denominator = (
            2 * derivative(current)^2
            - function(current) * second_derivative(current)
        )
        current = n(
            current - numerator / denominator,
            digits=digits,
        )
    return current


precision = 400
R = RealField(precision)

# 将当前连接显示的 y0 与迭代稳定后的 y 分别填入这里。
y0 = R("0.3241395720671662744547005650...")
limit = R("0.2798370212298224369684872811...")

f(x) = (x^3 * y0^4)^(1 / 7) - limit
recovered = halley(f, R("0.7"), precision)

# x0 的小数部分就是 flag 大整数的十进制数字。
flag_integer = int(str(recovered)[2:])
print(long_to_bytes(flag_integer))
```

部署源码中的真实 flag 为：

```text
N0PS{fl04T_nUm8eR_RePre53nT_rEal_v4Lue_Wi7h_d3c1mal5}
```

仓库自带 `writeup.md` 末尾误写成了另一题 `n0psichu` 的 flag；这里根据 `src/docker/flag.py` 和本题的实数恢复机制进行了修正。

## 方法总结

处理非线性迭代时，取对数常能把乘法和幂运算转成线性递推。本题真正的突破口是找出不变量 $3\log x+4\log y$，再利用共同极限恢复初值。flag 被编码进高精度小数后，数值误差会直接破坏末尾字节，因此求根、字符串转换和整数恢复的全过程都必须保持足够精度。
