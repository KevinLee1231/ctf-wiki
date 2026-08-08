# miniLCTF 2024 Ezfactor Writeup

## 题目简述

题目给出 $n=pq$，其中 $p,q$ 均为 768 位素数，同时泄露 `gift = p >> 360`。另外存在两组不同的整数解

$$x_1^2+e y_1^2=n,\qquad x_2^2+e y_2^2=n,$$

AES 密钥和 IV 分别由 $x_1+x_2+y_1+y_2$ 与异或组合导出。需要先补全 $p$、分解 $n$，再构造 $n$ 的两组二次型表示。

## 解题过程

### Coppersmith 补全素数

写成

$$p=(\text{gift}\ll 360)+x_0,\qquad 0\le x_0<2^{360}.$$

因为 $p\mid n$ 且 $p\approx n^{1/2}$，可对一元首一多项式 $f(x)=(\text{gift}\ll360)+x$ 求模未知因子的短根。题目参数下，以下配置可以稳定找到根：

```sage
PR.<x> = PolynomialRing(Zmod(n))
f = (gift * 2**360 + x).monic()
roots = f.small_roots(
    X=2**360,
    beta=0.49,
    epsilon=0.01498519292011843,
)
assert roots
x0 = int(roots[0])
p = gift * 2**360 + x0
assert n % p == 0
q = n // p
```

本地求得低位根：

```text
588731740812295178064410843438594041835770881962883653529238359983430223068069894979182679831746545826511947
```

恢复出的 $p,q$ 均为 768 位素数。

### 从素因子构造两组表示

分别求

$$p=a_p^2+e b_p^2,\qquad q=a_q^2+e b_q^2.$$

可用 Cornacchia 算法：先求 $r^2\equiv-e\pmod p$，再用欧几里得过程把余数降到 $\sqrt p$，检查剩余量能否写成 $e$ 乘完全平方；对 $q$ 同理。

由 Brahmagupta 恒等式，两组不同符号的组合给出 $n=pq$ 的两种表示：

```python
xp, yp = cornacchia(e, p)
xq, yq = cornacchia(e, q)

rep1 = (xp*xq - e*yp*yq, xp*yq + xq*yp)
rep2 = (xp*xq + e*yp*yq, xp*yq - xq*yp)

for xx, yy in (rep1, rep2):
    assert xx*xx + e*yy*yy == n

x1, y1 = map(abs, rep1)
x2, y2 = map(abs, rep2)
key = long_to_bytes(x1 + x2 + y1 + y2)[:16]
iv = long_to_bytes((x1 ^ x2) + (y1 ^ y2))[:16]
print(unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16))
```

本地验证两组表示均严格满足方程，导出的密钥和 IV 为：

```text
key = eb28913f04752bccd15530ebbece38d1
iv  = d3164ebef3409232ceaacec33eada691
```

最终得到：

```text
miniLCTF{!@#$s0_eazy_f4ct0r!@#$}
```

## 方法总结

本题分成两个独立的数论阶段：先利用已知高位对 $p$ 做 Coppersmith 部分密钥恢复，再利用 Cornacchia 算法与二次型乘法恒等式构造两组表示。只完成因数分解还不足以解密，必须继续利用 $x^2+ey^2$ 的结构恢复密钥材料。
