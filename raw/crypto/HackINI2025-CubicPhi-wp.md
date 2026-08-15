# CubicPhi

## 题目简述

题目给出 RSA 公钥参数 $N$、$e=65537$、密文 $C$，并额外泄露 $\varphi(N^3)$。关键不是分解大整数 $N$，而是利用欧拉函数在同一整数幂上的关系，直接恢复 $\varphi(N)$，进而求出私钥指数。

## 解题过程

对任意正整数 $N$ 和整数 $k\geq 1$，都有

$$
\varphi(N^k)=N^{k-1}\varphi(N).
$$

因此本题泄露量满足

$$
\varphi(N^3)=N^2\varphi(N),\qquad
\varphi(N)=\frac{\varphi(N^3)}{N^2}.
$$

得到 $\varphi(N)$ 后，按标准 RSA 解密流程计算

$$
d\equiv e^{-1}\pmod{\varphi(N)},\qquad
m\equiv C^d\pmod N.
$$

求解代码如下；题目给出的长整数可直接填入对应变量：

```python
from Crypto.Util.number import inverse, long_to_bytes

N = ...
phi_N3 = ...
e = 65537
C = ...

phi_N = phi_N3 // (N**2)
assert phi_N * N**2 == phi_N3

d = inverse(e, phi_N)
m = pow(C, d, N)
print(long_to_bytes(m))
```

脚本输出：

```text
shellmates{S1mpl3_DiV1$10n_1$_tHe_k3Y}
```

## 方法总结

- 核心技巧：由 $\varphi(N^3)=N^2\varphi(N)$ 从高次模数的欧拉函数泄露中恢复 RSA 私钥所需的 $\varphi(N)$。
- 识别信号：RSA 题同时给出 $N$ 与 $\varphi(N^k)$ 时，应先检查是否能直接除以 $N^{k-1}$，无需分解 $N$。
- 复用要点：恢复后先用整除关系做一致性校验，再计算模逆和明文，避免把抄错的长整数当成密码攻击失败。
