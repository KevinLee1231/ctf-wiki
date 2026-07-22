# RSA leak

## 题目简述

题目先随机生成两个约 256 位整数 $a,b$，再构造

$$
p=\operatorname{next\_prime}(a^4+r_p),\qquad
q=\operatorname{next\_prime}(b^4+r_q),
$$

其中传入 `next_prime` 的初始偏移小于 $2^{24}$。程序随后把偏移更新为真实差值

$$
r_p=p-a^4,\qquad r_q=q-b^4,
$$

并用 $n=pq$、$e=65537$ 加密 flag。额外泄露为另取两个 64 位素数生成模数 $N_0$，输出

$$
L\equiv r_p^e+r_q^e+\mathtt{0xdeadbeef}\pmod {N_0}.
$$

目标是先从约 24 位的两个偏移中恢复 $r_p,r_q$，再利用 $p,q$ 都极接近完全四次幂的结构分解 RSA 模数。

## 解题过程

### 1. 中途相遇恢复两个偏移

把常数移到等式另一侧：

$$
r_p^e\equiv L-\mathtt{0xdeadbeef}-r_q^e\pmod {N_0}.
$$

预计算搜索范围内所有 $x^e\bmod N_0$，建立“模幂值到 $x$”的哈希表；再枚举另一侧的 $y$，查询

$$
(L-\mathtt{0xdeadbeef}-y^e)\bmod N_0
$$

是否在表中。时间和空间复杂度均约为 $O(2^{24})$，远小于直接枚举两个偏移的 $O(2^{48})$。

需要注意，源码限制的是送入 `next_prime` 前的随机偏移，真实的 $r_p,r_q$ 还包含随后跨过的素数间隔，未必严格小于 $2^{24}$。实际搜索应在 $2^{24}$ 后预留一小段余量；若没有命中，再继续扩大。泄露式关于两者对称，因此恢复出的二元组还需要交换顺序并在后续步骤中验证。

### 2. 从模数取出 $ab$

令

$$
x=ab.
$$

展开 RSA 模数：

$$
\begin{aligned}
n&=(a^4+r_p)(b^4+r_q)\\
 &=x^4+r_q a^4+r_p b^4+r_pr_q.
\end{aligned}
$$

对本题的位数而言，后三项远小于相邻两个四次幂在 $x$ 附近的间隔

$$
(x+1)^4-x^4,
$$

所以给定实例满足

$$
x=\left\lfloor\sqrt[4]{n}\right\rfloor=ab.
$$

实现时应使用整数四次方根，并显式验证 $x^4\le n<(x+1)^4$，不能用浮点数计算千位整数的四次方根。

### 3. 解二次方程恢复 $a^4$ 与 $b^4$

记

$$
A=a^4,\qquad B=b^4,\qquad AB=x^4,
$$

以及

$$
S=n-x^4-r_pr_q.
$$

由模数展开式得到

$$
S=r_qA+r_pB=r_qA+\frac{r_px^4}{A}.
$$

两边乘以 $A$，得到关于 $A$ 的二次方程

$$
r_qA^2-SA+r_px^4=0.
$$

计算判别式

$$
\Delta=S^2-4r_pr_qx^4,
$$

并检查 $\Delta$ 是完全平方数。对两个根

$$
A=\frac{S\pm\sqrt\Delta}{2r_q}
$$

逐一验证：$A$ 必须为正整数和完全四次幂，$B=x^4/A$ 也必须为完全四次幂。若不成立，就交换 $r_p,r_q$ 再试。最终计算

$$
p=A+r_p,\qquad q=B+r_q
$$

并以 $pq=n$ 作最终确认。

核心求解逻辑如下；`bound` 应按前述素数间隔留出余量：

```python
from math import isqrt


def iroot4(value):
    """返回非负整数 value 的向下取整四次方根。"""
    low = 0
    high = 1 << ((value.bit_length() + 3) // 4)
    while low + 1 < high:
        middle = (low + high) // 2
        if middle**4 <= value:
            low = middle
        else:
            high = middle
    return low


def recover_offsets(modulus, leaked, exponent, bound):
    table = {}
    for rp in range(bound):
        table.setdefault(pow(rp, exponent, modulus), rp)

    target = (leaked - 0xDEADBEEF) % modulus
    for rq in range(bound):
        left = (target - pow(rq, exponent, modulus)) % modulus
        if left in table:
            yield table[left], rq


def factor_structured_rsa(n, rp, rq):
    x = iroot4(n)
    if not (x**4 <= n < (x + 1)**4):
        return None

    x4 = x**4
    s = n - x4 - rp * rq
    delta = s * s - 4 * rp * rq * x4
    if delta < 0:
        return None

    sqrt_delta = isqrt(delta)
    if sqrt_delta * sqrt_delta != delta:
        return None

    denominator = 2 * rq
    for numerator in (s + sqrt_delta, s - sqrt_delta):
        if numerator <= 0 or numerator % denominator:
            continue
        a4 = numerator // denominator
        if x4 % a4:
            continue
        b4 = x4 // a4
        a = iroot4(a4)
        b = iroot4(b4)
        if a**4 != a4 or b**4 != b4:
            continue
        p = a4 + rp
        q = b4 + rq
        if p * q == n:
            return p, q
    return None
```

对中途相遇得到的每个候选二元组，同时测试 `(rp, rq)` 与 `(rq, rp)`。分解成功后按普通 RSA 解密：

$$
\varphi(n)=(p-1)(q-1),\qquad
d\equiv e^{-1}\pmod{\varphi(n)},\qquad
m\equiv c^d\pmod n.
$$

最后把整数 $m$ 转回字节串即可得到 flag。

## 方法总结

本题的泄露不是直接给出 $p$ 或 $q$ 的比特，而是把两个小偏移的模幂相加。中途相遇先把 $2^{48}$ 的联合搜索降为约 $2^{24}$；随后利用 $n$ 紧邻完全四次幂得到 $ab$，再把剩余关系整理成关于 $a^4$ 的二次方程，精确恢复两个素因子。

关键实现细节有三点：真实偏移可能略大于 $2^{24}$；大整数开方必须使用整数算法；所有代数候选都要以“完全平方、完全四次幂、最终乘积等于 $n$”逐层校验，避免把模泄露碰撞或 $r_p,r_q$ 的顺序误判当成正确结果。
