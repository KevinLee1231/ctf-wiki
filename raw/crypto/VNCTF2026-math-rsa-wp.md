# math_rsa

## 题目简述

题目是一个 RSA 变形泄露。附件源码生成普通 RSA 参数 `p,q,n,e,phi,c`，但额外给出一个大整数 `k`，它和 `phi` 之间满足人为构造的代数关系。关键不是分解 `n`，而是把这个关系化简，从 `k` 中恢复 `phi`。

核心源码如下：

```python
p = getPrime(1024)
q = getPrime(1024)
n = p * q
e = 65537
phi = (p - 1) * (q - 1)

u = getPrime(16)
t = 2 * u
x = phi - 1
y = t + 1

assert((x**2 + 1) * (y**2 + 1) - 2 * (x - y) * (x * y - 1) == 4 * (k + x * y))

m = bytes_to_long(flag)
c = pow(m, e, n)
print("n =", n)
print("e =", e)
print("c =", c)
```

其中 `u` 是 16-bit prime，所以 `t = 2u` 的范围很小，可以枚举。`n,e,c,k` 是题目输出；`n,c,k` 都是长整数，不需要在 WP 正文中完整展开，真正有价值的是上面的构造关系。

## 解题过程

先把题目给出的方程移项，并用 Sage/SymPy 做因式分解。令：

```python
x, y = var("x,y")
f = (x**2 + 1) * (y**2 + 1) - 2 * (x - y) * (x * y - 1) - 4 * x * y
print(f.factor())
```

可以得到：

$$
(x+1)^2(y-1)^2
$$

题目源码里：

$$
x=\phi-1,\quad y=t+1
$$

因此：

$$
(x+1)^2(y-1)^2=\phi^2t^2
$$

而原方程等价于：

$$
(x+1)^2(y-1)^2=4k
$$

所以：

$$
\phi t = 2\sqrt{k}
$$

`t` 是 `2 * getPrime(16)`，大约落在 `[2^16, 2^17)`。枚举这个范围内能整除 `2 * sqrt(k)` 的候选，即可得到 `phi`：

```python
from Crypto.Util.number import long_to_bytes
import gmpy2

n = ...
e = 65537
c = ...
k = ...

phi_times_t = gmpy2.iroot(k, 2)[0] * 2

for t in range(2**16, 2**17):
    if phi_times_t % t != 0:
        continue
    phi = phi_times_t // t
    d = gmpy2.invert(e, phi)
    m = pow(c, d, n)
    flag = long_to_bytes(m)
    if b"VNCTF" in flag:
        print(flag)
        break
```

这里不需要恢复 `p`、`q`。只要 `phi` 正确，RSA 私钥指数 `d = e^{-1} mod phi` 就能直接解密。

## 方法总结

- 核心技巧：利用题目给出的代数恒等式恢复 RSA `phi`。
- 识别信号：RSA 题额外给出一个和未知小参数相关的大整数 `k`，源码中还有形如 `(x^2+1)(y^2+1)-...` 的构造式。
- 复用要点：不要急着分解 `n`；先对泄露方程符号化、因式分解，判断是否能把 `phi` 和小参数分离。若小参数来自短素数或短随机数，就优先枚举它。
