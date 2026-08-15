# Roots

## 题目简述

题目用同一 flag 分别构造两组 RSA：$N_1=pq$、$N_2=rs$，其中 $p$ 为 1024 位素数，$q,r,s$ 为 2048 位素数，并令 $r$ 为紧邻 $q$ 之后、且差值落在较小范围内的素数。公开数据包含 $N_1,N_2,C_1,C_2$。决定性弱点是小差值 $i=r-q$ 把两个本来不共享素因子的模数联系起来，可转化为模未知大因子的 Coppersmith 小根问题。

## 解题过程

令

$$
i=r-q.
$$

对多项式

$$
f_i(x)=N_1+ix
$$

代入未知小素数 $x=p$：

$$
f_i(p)=pq+ip=p(q+i)=pr.
$$

$pr$ 是 $N_1N_2=pqrs$ 的一个大因子。因此，当猜中较小的偶数差值 $i$ 时，$p$ 是多项式 $f_i(x)$ 在模 $N_1N_2$ 的一个未知大因子上的小根；同时题目生成代码给出 $p<2^{1024}$。官方 Sage solver 从可能的素数间隔开始枚举 $i$，对每个候选调用 `small_roots`：

```python
from sage.all import PolynomialRing, Zmod
from Crypto.Util.number import isPrime, inverse, long_to_bytes

N1 = ...
N2 = ...
C1 = ...
e = 65537

i = 512
while True:
    i += 2  # 大素数均为奇数，因此差值为偶数
    ring = Zmod(N1 * N2)
    PR = PolynomialRing(ring, "x")
    x = PR.gen()
    f = N1 + i * x

    roots = f.monic().small_roots(X=2**1024, beta=0.4)
    if roots:
        p = int(max(roots))
        if isPrime(p) and N1 % p == 0:
            break

q = N1 // p
phi = (p - 1) * (q - 1)
d = inverse(e, phi)
m = pow(C1, d, N1)
print(long_to_bytes(m))
```

`beta=0.4` 表示目标根对应的未知因子需要占总模数足够大的比例；本题中 $pr$ 约为 3072 位，而 $N_1N_2$ 约为 7168 位，参数与构造相符。恢复 $p$ 后无需再处理第二组密文，直接分解 $N_1$ 并解密 $C_1$ 即可。

最终得到：

```text
shellmates{n3VEr_UnD3R3$tImatE_$m4LL_ro0ts!!}
```

## 方法总结

- 核心技巧：把近邻素数关系 $r=q+i$ 改写成 $f_i(p)=pr$，再对组合模数的未知大因子求小根。
- 识别信号：两个 RSA 模数不共享素因子，但各自素因子之间存在已知的小偏移、线性关系或部分位关系时，应尝试构造在模乘积大因子上为零的多项式。
- 复用要点：枚举差值时利用奇素数差为偶数；小根候选必须再检查素性和整除性，不能仅凭 `small_roots` 返回非空就解密。
