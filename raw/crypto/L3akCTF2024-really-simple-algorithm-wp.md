# L3akCTF 2024 Really Simple Algorithm Writeup

## 题目简述

RSA 服务使用固定公开指数 $e=1337$，但每次菜单循环都会重新生成一对 1024 位素数和新的模数。请求加密 flag 时，明文始终相同而模数不断变化：

$$
c_i\equiv m^e\pmod{n_i}.
$$

相同明文、相同指数和大量两两互素模数正好满足 Håstad 广播攻击条件。

## 解题过程

连续请求 1337 次 flag 密文，收集：

$$
(n_1,c_1),(n_2,c_2),\ldots,(n_e,c_e).
$$

由于各模数独立生成，可用中国剩余定理求出唯一的：

$$
x\equiv c_i\pmod{n_i},
\qquad
0\leq x<\prod_i n_i.
$$

当样本足够多时，模数乘积大于 $m^e$，因此 CRT 得到的不只是一个剩余类，而是整数意义上的 $x=m^e$。随后计算精确的 1337 次整数根：

```python
from sage.all import CRT_list
import gmpy2

x = int(CRT_list(ciphertexts, moduli))
m, exact = gmpy2.iroot(x, 1337)
assert exact
print(long_to_bytes(int(m)))
```

必须检查 `exact`，否则取整后的近似根可能被误当作明文。官方 solver 恢复的 flag 为：

```text
L3AK{H4sTAD5_bR0aDc45T_4TtacK_1s_pr3tTy_c0ol!}
```

## 方法总结

- 核心技巧：对相同 RSA 明文在不同模数下的密文做 CRT，再取精确 $e$ 次整数根。
- 识别信号：服务反复更换模数，却固定指数并加密同一条未随机填充消息，这是广播攻击的直接信号。
- 复用要点：样本数需要让 $\prod n_i>m^e$；实际利用时还应检查模数是否两两互素以及整数根是否精确。
