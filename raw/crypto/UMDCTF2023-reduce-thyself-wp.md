# Reduce Thyself

## 题目简述

服务为每个连接生成 RSA 参数，公开 $n$、$e=65537$ 和 flag 密文

$$
c_f=m^e\bmod n.
$$

解密 oracle 拒绝原始 $c_f$，并且只接受其整数代表是素数的其他密文。题面指向 Boneh 与 Shoup 教材中关于 RSA 随机自归约性的章节；其关键结论是 RSA 的乘法同态允许用随机因子把一个目标实例变成另一个实例，再从后者的解还原前者。本题已经把这一重要条件完整暴露在 oracle 中。

## 解题过程

随机选择 $r\in\mathbb Z_n^\*$，构造

$$
c' = c_f\cdot r^e\bmod n.
$$

若把 $c'$ 看作 $[0,n)$ 内的整数后恰好为素数，并且 $c'\ne c_f$，服务就会解密。RSA 的乘法性质给出：

$$
\begin{aligned}
\operatorname{Dec}(c')
&=(c_f\cdot r^e)^d\\
&=m^{ed}\cdot r^{ed}\\
&=mr\pmod n.
\end{aligned}
$$

所以从 oracle 返回值 $m'=mr\bmod n$ 中乘以 $r^{-1}$ 即可恢复 flag：

```python
while True:
    r = random.randrange(2, n)
    if math.gcd(r, n) != 1:
        continue

    blinded = flag_ct * pow(r, e, n) % n
    if blinded != flag_ct and isPrime(blinded):
        break

response = oracle_decrypt(blinded)
flag_int = response * inverse(r, n) % n
print(long_to_bytes(flag_int).decode())
```

一个随机的 2048 位整数为素数的概率约为 $1/\ln n$，通常尝试几千次即可命中；无需分解模数。得到：

```text
UMDCTF{s3lf_r3duc!bility_1s_n0t_ju5t_f0r_DH_97912837923}
```

## 方法总结

- 核心技巧：用 RSA 乘法同态对目标密文做随机盲化，使其满足 oracle 的“密文整数为素数”过滤条件，再去盲化明文。
- 识别信号：RSA 解密 oracle 只按密文表示做白名单或黑名单过滤，却没有使用 OAEP 等非延展编码时，应测试乘法自归约。
- 复用要点：随机因子必须在 $\mathbb Z_n^\*$ 中，且服务器收到的是模 $n$ 后的密文代表；过滤条件应在取模之后检查。
