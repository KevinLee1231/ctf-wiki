# keysmith

## 题目简述

服务给出一对整数 $(m,s)$，其中 $s=m^{65537}\bmod n_0$，但不公开原模数 $n_0$。参赛者可以自行提交新的 $p,q,e$；服务在 $n=pq$ 上计算 $d=e^{-1}\bmod\varphi(n)$，并要求同时满足

$$
m^e\equiv s\pmod n,\qquad s^d\equiv m\pmod n.
$$

因此目标不是找回原 RSA 私钥，而是为已知映射 $(m,s)$ 重新“铸造”一套可通过检查的 RSA 参数。

## 解题过程

选择满足 $p-1$、$q-1$ 光滑的两个新素数，并确保 $m,s$ 在相应乘法群中可逆且生成足够大的子群。这样可以快速求出

$$
d_p=\log_s m\pmod{p-1},\qquad d_q=\log_s m\pmod{q-1}.
$$

用 CRT 合并得到 $d$，再令 $e=d^{-1}\bmod\varphi(pq)$。官方脚本的关键流程为：

```sage
# getSmooth() 构造 p-1、q-1 可完全分解的素数，并在失败时重试。
p, p_factors = getSmooth(512)
q, q_factors = getSmooth(512, p_factors)

dp = discrete_log(Mod(m, p), Mod(s, p), p - 1)
dq = discrete_log(Mod(m, q), Mod(s, q), q - 1)
d = crt([dp, dq], [p - 1, q - 1])
phi = (p - 1) * (q - 1)
e = inverse_mod(d, phi)

n = p * q
assert power_mod(m, e, n) == s
assert power_mod(s, d, n) == m
```

把通过本地双向断言的 $p,q,e$ 发送给服务即可得到：

```text
tjctf{lock-smith_289378972359}
```

## 方法总结

- 当验证器允许攻击者选择群参数时，应围绕已知输入输出构造一个容易求解的新群，而不是执着于恢复原参数。
- 让 $p-1$、$q-1$ 光滑可把离散对数交给 Pohlig–Hellman；CRT 则把两个素数模下的指数关系拼回 RSA 模数。
- 必须检查 $d$ 在 $\varphi(n)$ 下可逆、CRT 同余相容，以及两条服务端等式都成立；不满足就重新生成参数。
