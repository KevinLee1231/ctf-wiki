# RSA

## 题目简述

题目把明文 $m$、RSA 素数 $p,q$ 组合进一个 $4\times4$ 矩阵，再在模 $n=pq$ 下计算 $M^e$。源码中矩阵为四元数的左乘矩阵表示：

```sage
M = matrix(Zmod(n), [
    [m, -m-p-q, -m-2*p, 2*q-m],
    [m+p+q,  m, 2*q-m,  m+2*p],
    [m+2*p,  m-2*q,  m, -m-p-q],
    [m-2*q, -m-2*p,  m+p+q,  m]
])
enc = M**e
```

## 解题过程

### 关键机制

四元数 $q=a+bi+cj+dk$ 可表示为矩阵：

$$
\begin{pmatrix}
a & -b & -c & -d \\
b & a & -d & c \\
c & d & a & -b \\
d & -c & b & a
\end{pmatrix}
$$

外链论文的关键结论是：四元数的 $n$ 次幂可以写成关于 $a$ 和 $S=-(b^2+c^2+d^2)$ 的组合。设：

$$
X=\sum_{i=0}^{\lfloor(n-1)/2\rfloor}{n\choose n-2i-1}a^{n-2i-1}S^i
$$

则幂后的虚部满足：

$$
b_n=bX,\quad c_n=cX,\quad d_n=dX
$$

因此可从密文矩阵的若干位置构造出 $p$ 或 $q$ 的倍数，进而分解 $n$。

参考 URL：https://www.scirp.org/journal/paperinformation.aspx?paperid=116312

### 求解步骤

从 `enc` 提取：

```python
an = enc[0][0]
bn = enc[1][0]
cn = enc[2][0]
dn = enc[3][0]
```

利用官方给出的线性组合：

```python
from Crypto.Util.number import *

qx = (2 * bn - cn - dn) * pow(4, -1, n)
q = GCD(qx, n)
p = n // q
X = (cn - bn) * inverse(p - q, n) % n
b_ = int(bn * inverse(X, n) % n)
m = b_ - p - q
print(long_to_bytes(m))
```

得到 flag：

```text
WMCTF{QU4t3rni0n_4nd_Matr1x_4r3_4un}
```

## 方法总结

- 题目不是普通 RSA，而是四元数矩阵幂。
- 关键是把矩阵还原成四元数虚部，利用 $b_n=bX,c_n=cX,d_n=dX$ 的共同因子关系。
- 分解出 $n$ 后，按源码中 $b=m+p+q$ 的关系恢复明文。
