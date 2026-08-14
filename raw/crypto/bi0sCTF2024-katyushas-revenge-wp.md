# bi0sCTF 2024 - Katyusha's Revenge

## 题目简述

题目生成 2048 位 RSA 模数 $n=pq$ 与一个 132 位素数 $e$，泄漏私钥指数在模 $p-1$ 下的值 $d_p$ 的低 892 位。flag 没有使用普通 RSA 加密，而是作为指数嵌入 Damgard-Jurik 密文：

$$
c=3^m r^{n^4}\pmod{n^5}.
$$

解题分为两步：利用 $d_p$ 的低位泄漏和二元 Coppersmith 分解 $n$，再实现参数 $s=5$ 的 Damgard-Jurik 解密，分别消去底数 3 并还原消息 $m$。

## 解题过程

### 把 $d_p$ 泄漏写成因子关系

设

$$
d_p=2^{892}x+d_{p,\mathrm{low}},\qquad |x|<2^{132}.
$$

因为 $ed\equiv1\pmod{\varphi(n)}$，所以

$$
ed_p-1=t(p-1)
$$

对某个约 132 位的整数 $t$ 成立。$e$ 与 $d_p$ 都是奇数，因此 $ed_p-1$ 为偶数；结合素数 $p$ 为奇数，官方 solver 将 $t$ 参数化为奇数 $2k+1$。展开可得

$$
ed_p-1=(2k+1)(p-1),
$$

$$
ed_p+2k=(2k+1)p.
$$

于是多项式

$$
f(x,k)=1-(2k+1)-e\left(2^{892}x+d_{p,\mathrm{low}}\right)
$$

在真实根处是 $p$ 的倍数，因而同时满足 $f(x,k)\equiv0\pmod p$，其中 $p$ 是公开模数 $n$ 的大因子。

### 二元 Coppersmith 分解 $n$

官方解法使用

$$
|x|<2^{132},\qquad |k|<2^{131}
$$

作为根界，取格参数 $m=13$ 建立二元 Coppersmith 格，并用 `flatter` 完成格约简。由短向量恢复整数多项式后，通过 Groebner 基求出候选 $(x,k)$。

无需先显式恢复 $p$ 的表达式，只要把候选代回原多项式：

```python
dp = (1 << 892) * x + dp_low
value = 1 - (2 * k + 1) - e * dp
p = gcd(value, n)
q = n // p
assert p * q == n
```

这是因为真实 `value` 含有非平凡因子 $p$。若 `gcd` 只得到 1 或 $n$，说明低根候选或参数不正确。

### 解开 Damgard-Jurik 密文

Damgard-Jurik 是 Paillier 的推广，参数 $s=5$ 时工作模数为 $n^5$，明文位于 $\mathbb Z_{n^4}$。题目密文中的 $r^{n^4}$ 是随机化因子；知道分解后，可用 $\varphi(n)$ 消去它，并逐层从模 $n^2,n^3,\ldots,n^5$ 的结果中恢复离散消息指数。

官方 `decrypt(ct, phi, n, s)` 函数先计算

$$
a=ct^{\varphi(n)}\pmod{n^s},
$$

再按 $n$ 的幂逐层修正二项式展开，最终得到密文底数对应的消息。由于题目使用 $3^m$ 而不是标准生成元直接编码，分别计算

```python
dm = decrypt(c, phi, n, 5)
d3 = decrypt(3, phi, n, 5)
m = dm * inverse(d3, n**4) % (n**4)
flag = long_to_bytes(m)
```

解密映射在消息加法群上是线性的，因此 $D(3^m)=mD(3)$；乘上 $D(3)^{-1}$ 就能消去底数带来的比例因子。

## 方法总结

本题把 RSA 部分密钥泄漏与 Damgard-Jurik 组合在一起。$d_p$ 的已知低位通过 $ed_p-1=t(p-1)$ 直接关联到素因子；两个未知量都只有约 132 位，适合二元 Coppersmith。分解模数后还不能照搬 RSA 解密，而要按 $n$ 的幂逐层反演 Damgard-Jurik，并额外消除题目选择的底数 3。
