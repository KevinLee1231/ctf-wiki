# EZC

## 题目简述

题目在 secp256k1 的素数域上使用曲线 $y^2=x^3+ax+b$，却把 flag 的前后两半直接编码为曲线参数 `a` 和 `b`，并公开曲线上的两个点。两个点正好提供两条关于 `a`、`b` 的线性同余方程，可以唯一恢复参数并拼回 flag。

## 解题过程

### 从两个点建立线性方程

对每个公开点 $(x_i,y_i)$，移项可得：

$$
t_i=y_i^2-x_i^3\equiv ax_i+b\pmod p
$$

两式相减：

$$
a\equiv(t_1-t_2)(x_1-x_2)^{-1}\pmod p
$$

随后代回求：

$$
b\equiv t_1-a x_1\pmod p
$$

附件中的两行均为 `Point(x, y)`，可以直接解析：

```python
import re
from Crypto.Util.number import long_to_bytes

p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
raw = open("out.txt", "r", encoding="utf-8").read()
numbers = [int(value) for value in re.findall(r"\d+", raw)]
(x1, y1), (x2, y2) = numbers[:2], numbers[2:4]

t1 = (y1 * y1 - x1**3) % p
t2 = (y2 * y2 - x2**3) % p
a = (t1 - t2) * pow(x1 - x2, -1, p) % p
b = (t1 - a * x1) % p

flag = long_to_bytes(a) + long_to_bytes(b)
print(flag)
```

恢复出的两段分别是 `shellmates{3lipt1` 和 `c_curv3s_4r3_c00l}`，拼接后得到：

```text
shellmates{3lipt1c_curv3s_4r3_c00l}
```

## 方法总结

- 核心技巧：将公开曲线点代入曲线方程，把未知参数转化为有限域上的二元一次方程组。
- 识别信号：题目隐藏的是曲线参数而非离散对数，并且给出两个以上点时，应优先联立曲线方程。
- 复用要点：所有减法和乘法都应在模 $p$ 下进行；求逆前需确认 $x_1\not\equiv x_2\pmod p$。
