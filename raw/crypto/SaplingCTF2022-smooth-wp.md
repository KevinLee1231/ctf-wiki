# Smooth

## 题目简述

服务返回 $g^m\bmod p$，其中 $m$ 是 flag 转成的整数，$p$ 没有直接给出。模数 $p-1$ 的绝大部分因子都很小，只有一个约 256 位的大因子，因此可以先从 oracle 输出恢复 $p$，再在各小素数幂子群中执行 Pohlig-Hellman，最后用 CRT 合并出足够多的 flag 低位。

## 解题过程

查询若干已知指数 $a_i$，得到 $y_i=g^{a_i}\bmod p$。每个整数差 $g^{a_i}-y_i$ 都是 $p$ 的倍数：

~~~python
from math import gcd

samples = [(2, oracle(2)), (3, oracle(3)), (5, oracle(5))]
p = 0
for exponent, value in samples:
    p = gcd(p, pow(g, exponent) - value)
~~~

必要时继续查询并取 gcd，去掉额外公共倍数。分解 $p-1$ 后，跳过无法处理的 256 位大因子，只对其余素数幂 $q_i^{e_i}$ 求离散对数。把生成元和目标投影到对应子群：

$$
g_i=g^{(p-1)/q_i^{e_i}},\qquad
h_i=h^{(p-1)/q_i^{e_i}},
$$

即可得到 $m\bmod q_i^{e_i}$。将所有余数用 CRT 合并，得到：

$$
m\equiv m_0\pmod M.
$$

小因子乘积 $M$ 约有 302 位，而 flag 约 313 位，所以只剩大约 11 位未知。枚举 $m=m_0+kM$，检查字节串是否包含 maple{：

~~~python
for k in range(1 << 12):
    candidate = m0 + k * M
    raw = long_to_bytes(candidate)
    if b"maple{" in raw:
        print(raw)
        break
~~~

恢复结果为：

~~~text
maple{sm00th_3n0ugh_t0_f!nd_th3_dl0g!!}
~~~

## 方法总结

离散对数的难度取决于群阶的最大素因子，而不只是模数位数。若 $p-1$ 足够平滑，Pohlig-Hellman 会把一个大问题拆成许多小问题。本题还展示了一个通用技巧：只恢复指数模一个很大的 $M$ 也可能足够，因为消息格式和较小剩余空间能唯一确定整数。
