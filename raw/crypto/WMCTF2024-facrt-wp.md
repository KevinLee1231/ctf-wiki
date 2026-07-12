# FACRT

## 题目简述

题目是 RSA-CRT 故障攻击。源码随机生成 $p,q$，对 20 个随机消息做 CRT 签名，但在计算 $s_q=m^{d_q}\bmod q$ 后人为清零高 32 位：

```python
sq.append(pow(m[i], dq, q))
sq_.append(int("0" * 32 + bin(pow(m[i], dq, q))[2:][32:], 2))
sp.append(pow(m[i], dp, p))
s.append(int(sq_[i] + q * (((sp[i] - sq[i]) * iq) % p)))
```

同时输出 RSA 加密后的 flag `c=pow(flag,e,n)`。

## 解题过程

### 关键机制

外链论文 ePrint 2024/1125 的关键结论是：如果 CRT 签名中一侧模 $q$ 的结果只有低位保留，即 $s_q^*<q/2^\ell$，多组错误签名可转化为 Partial Approximate Common Divisor 问题，进而恢复 $q$ 并分解 $N$。

参考 URL：https://eprint.iacr.org/2024/1125.pdf

题目中 $\ell=32$，$q$ 为 512 bit，因此小误差约为：

$$
\rho=512-32=480
$$

错误签名可写成：

$$
s_i=r_i+qp_i
$$

其中 $r_i=s_q^*$ 很小。因为 $N=qp$，构造格后期望短向量形如：

$$
(2^\rho p,\;p r_1,\;p r_2,\ldots,p r_t)
$$

恢复 $p$ 或 $q$ 后即可正常 RSA 解密 flag。

### 求解步骤

1. 读取输出的 $N$、20 个错误签名 $s_i$ 和密文 $c$。
2. 根据 Partial ACD 构造格，把 $s_i$ 当作接近 $q$ 倍数的数：

$$
s_i\equiv r_i\pmod q,\quad |r_i|<2^\rho
$$

3. LLL/BKZ 求短向量，恢复 $p$ 或 $q$。
4. 计算：

```python
phi = (p - 1) * (q - 1)
d = inverse(e, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

## 方法总结

- 故障不是直接给一组错误签名求 gcd，而是多组“小余数”签名构成 Partial ACD。
- 清零高 32 位让 $s_q^*$ 足够小，满足格攻击条件。
- 论文外链的核心信息已归纳为：多组 $s_i=r_i+qp_i$ 且 $r_i$ 小，可恢复公共因子 $q$。
