# ChristmasZone

## 题目简述

题目把 flag 整数先送入一个由线性同余序列生成系数的五次多项式，再把结果拆成实部、虚部，按复数乘法模 $n=pq$ 做指数运算。公开量包括四组多项式取值、密文三元组 `(re, im, n)` 和 `gift`。

源码中的 `gift` 不是普通提示值。令 $s=p+q$、$N=pq$，其模数为

$$
\Phi=N^2+N+1+(N+1)s+p^2+q^2
=N^2+Ns+s^2-N+s+1.
$$

程序取一个约 400 位的素数 `mistletoe`，返回

$$
gift\equiv mistletoe^{-1}\pmod{\Phi}.
$$

因此可以构造关于小未知量 $s$ 和商 $k$ 的二元模方程，用 Coppersmith/LLL 恢复 $p+q$，进而分解 $N$。

## 解题过程

### 从 gift 恢复 p+q

由逆元关系存在整数 $k$，满足

$$
gift\cdot mistletoe=1+k\Phi.
$$

在模 `gift` 的环上消去 `mistletoe`，得到

$$
f(s,k)=k\left(N^2+Ns+s^2-N+s+1\right)+1\equiv0
\pmod{gift}.
$$

$s=p+q$ 约为 513 位，$k$ 受 400 位 `mistletoe` 限制。以界
$|s|<2^{513}$、$|k|<2^{400}$ 构造移位多项式格并做 LLL，即可取得小根。核心 Sage 代码如下；`small_roots` 是常见的多元 Coppersmith 格构造函数：

```sage
R = Integers(gift)
PR.<s, k> = PolynomialRing(R)
f = k * (N^2 + N*s + s^2 - N + s + 1) + 1

s, k = small_roots(f, bounds=(2^513, 2^400), m=3, d=4)[0]

P.<x> = PolynomialRing(ZZ)
(p, _), (q, _) = (x^2 - ZZ(s)*x + N).roots()
assert p*q == N
```

这里使用的是 $x^2-sx+N=0$，所以恢复出的根必须重新检查乘积，不能只凭 LLL 返回值继续计算。

### 逆复数幂运算

题目的乘法为

$$
(a,b)\otimes(c,d)=(ac-bd,ad+bc)\pmod N.
$$

源码给出的群阶倍数为

$$
\varphi_G=(p^2-1)(q^2-1).
$$

指数固定为 $e=65537$。计算 $d=e^{-1}\bmod\varphi_G$，再对密文做同一套平方乘，就能恢复加密前被拆开的多项式值：

```python
def mul_pair(P, Q, mod):
    a, b = P
    c, d = Q
    return ((a*c - b*d) % mod, (a*d + b*c) % mod)

def power(P, e, mod):
    out = (1, 0)
    while e:
        if e & 1:
            out = mul_pair(out, P, mod)
        P = mul_pair(P, P, mod)
        e >>= 1
    return out

order = (p*p - 1) * (q*q - 1)
d = inverse_mod(0x10001, order)
m1, m2 = power((re, im), d, N)
encoded = long_to_bytes(int(m1)) + long_to_bytes(int(m2))
```

### 还原前置多项式

`encoded` 是 `Function_function(flag_integer)`，不是可以无条件当作 flag 的普通 RSA 明文。生成器状态满足

$$
x_{i+1}=a x_i+b\pmod p,
$$

而公开的四组 `(input, output)` 都由同一组 $(a,b,x_0)$ 生成。把四个等式与递推式写入 $\mathbb F_p$ 上的多项式环，用 Gröbner basis 解出 $(a,b,x_0)$，即可重新生成六个系数 $c_0,\ldots,c_5$。随后求

$$
\sum_{i=0}^{5}c_iX^i-\operatorname{bytes\_to\_long}(encoded)=0
\pmod p.
$$

将各个根转为字节，只保留满足 `SCTF{...}` 格式且重新代入原函数后与完整整数值一致的候选。最终得到：

```text
SCTF{winerwienerCKdinner}
```

## 方法总结

这道题有两层，不能在分解 $N$ 后就停止。第一层利用“小逆元对应小商”的关系，把 `gift` 转成关于 $p+q$ 的二元小根问题；恢复 $p+q$ 后通过二次方程分解模数，并用复数乘法群的阶逆指数。第二层再从四组函数值恢复 LCG 参数和多项式系数，反解原始 flag。可靠的验证顺序是：检查 $pq=N$、检查复数幂正向重加密、检查候选根的完整多项式值，最后再检查 flag 格式。
