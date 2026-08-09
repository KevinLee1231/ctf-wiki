# cubic

## 题目简述

服务先生成两个 1024 位特殊素数 $p,q$，令 $N=pq$。私钥指数 $d$ 只有约 350 位，并允许在同一个 $N$ 上反复生成新的 $(e,d)$：

$$
ed\equiv1\pmod{\varphi(N)}.
$$

取得两组公钥 $(e_1,N)$、$(e_2,N)$ 后，服务又把 flag 分成两个整数，放入自定义二元群中，以其中一个公开指数加密；实际运算模数为 $pqr$，其中额外素数 $r$ 会随密文公开。主线因此分成两步：先利用“同模数 + 两个小私钥指数”恢复 $\varphi(N)$，再按自定义群的阶解密。

## 解题过程

### 用两个小 d 恢复 phi

对两组密钥分别有

$$
e_1d_1-k_1\varphi(N)=1,
\qquad
e_2d_2-k_2\varphi(N)=1.
$$

$d_1,d_2$ 只有约 350 位，远小于约 2048 位的 $N$。官方解法把两组近似整除关系同时编码进四维格。使用 $x=0.355$ 缩放小变量后，对下列基做 LLL：

```python
B = Matrix(ZZ, [
    [1, -N,   0,    N^2],
    [0,  e1, -e1, -e1*N],
    [0,   0,  e2, -e2*N],
    [0,   0,   0,   e1*e2],
])

x = 0.355
D = diagonal_matrix(ZZ, [N, floor(N^(1/2)), floor(N^(1+x)), 1])
L = (B * D).LLL()
coeff = L[0] * (B * D).inverse()
phi = ZZ(e1 * coeff[1] // coeff[0])
```

LLL 的第一短向量给出两条小私钥关系的整数线性组合，从中还原 $\varphi(N)$。必须用两项硬校验排除符号或短向量选择错误：

```python
d1 = inverse_mod(e1, phi)
assert d1.nbits() <= 360
assert (e1 * d1 - 1) % phi == 0
```

已知一组合法的 $(e_1,d_1,N)$ 后，可以使用标准“由私钥指数分解 RSA 模数”的随机算法。令 $K=e_1d_1-1=2^t u$，随机选 $g$，不断计算 $g^{K/2^j}\bmod N$；一旦出现非平凡平方根，`gcd(x-1, N)` 就会给出 $p$ 或 $q$。

### 计算自定义群阶并解密

flag 加密并不是普通的 $m^e\bmod N$。源码的 `add(P, Q, mod)` 定义了一个二元群运算，`myPower` 是该运算下的平方乘。对模数 $pqr$，可使用的群阶为

$$
\Phi_G=(p^2+p+1)(q^2+q+1)(r^2+r+1).
$$

服务输出密文点 $C=(c_x,c_y)$ 和 $r$。若 $\gcd(e_2,\Phi_G)=1$，计算 $d=e_2^{-1}\bmod\Phi_G$，再调用与题目完全相同的群运算：

```python
modulus = p * q * r
group_order = (p*p + p + 1) * (q*q + q + 1) * (r*r + r + 1)

if gcd(e2, group_order) != 1:
    raise ValueError("本轮指数不可逆，重新连接取一组参数")

d = inverse_mod(e2, group_order)
m1, m2 = myPower((cx, cy), int(d), modulus)
flag = long_to_bytes(int(m1)) + long_to_bytes(int(m2))
print(flag)
```

服务允许重新连接，因此遇到指数与群阶不互素的轮次直接重取参数即可。最终字节串应以 `SCTF{` 开头；仓库没有保留实际 `FLAG.py`，所以不把缺失的最终字符串写成已验证结果。

## 方法总结

本题表面上既有特殊素数又有自定义群，真正先破坏密钥安全的是同一 $N$ 下重复给出公钥且每次私钥指数都很小。两条 $e_id_i-k_i\varphi(N)=1$ 关系足以用低维 LLL 恢复 $\varphi(N)$，随后即可分解 $N$。解密阶段不能套普通 RSA 的 $(p-1)(q-1)$，而要从源码采用自定义群的阶，并把公开的第三个素数 $r$ 纳入模数和阶。仓库缺少 flag 文件时，WP 应保留可复现的判定条件，而不是补造最终明文。
