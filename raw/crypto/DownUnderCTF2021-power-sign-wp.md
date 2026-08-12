# DownUnderCTF 2021 - power sign

## 题目简述

服务实现了类似 Rabin 的签名：公钥为 $N=pq$，其中 $p\equiv q\equiv3\pmod4$；私钥是 $p,q$。对消息 $M$ 和递增参数 $u$ 计算自定义哈希 $H(M,u)$，找到它同时为模 $p$、模 $q$ 二次剩余时，返回平方根 $x$。验证条件只有：

$$
x^2\equiv H(M,u)\pmod N.
$$

哈希在扩域 $K=\mathbb F_{r^{15}}$ 中工作，其中 $r$ 是大于 $N$ 的下一个素数。消息的 $r$ 进制数字被映射成 $K$ 中的元素 $h$，再计算 $(h+uz)^{r^3}$，输出结果的常数项。服务允许选择一条消息让服务器签名，然后要求伪造随机消息。

## 解题过程

若能让签名 oracle 对已知平方 $s^2\bmod N$ 求平方根，服务器可能返回四个根中的非平凡根 $x\not\equiv\pm s\pmod N$。此时：

$$
s^2\equiv x^2\pmod N
\Longrightarrow
(s-x)(s+x)\equiv0\pmod N,
$$

所以 $\gcd(s-x,N)$ 会泄露 $p$ 或 $q$。

难点是消息必须满足 $N^3<M<N^{15}$，且经过自定义哈希后仍输出 $s^2\bmod N$。由于 $3\mid15$，$K$ 包含子域 $E=\mathbb F_{r^3}$；子域中所有元素都满足 $a^{r^3}=a$，即 Frobenius 映射在 $E$ 上是恒等映射。

取 $E$ 的生成元 $z_E$，将它嵌入 $K$，设其常数项为 $e_0$。构造：

$$
k=e_0^{-1}(s^2\bmod N)z_E-z.
$$

把 $k$ 的系数按服务端相反的 $r$ 进制顺序编码为整数消息。签名函数第一次尝试时 $u=1$，于是内部元素变成：

$$
k+z=e_0^{-1}(s^2\bmod N)z_E\in E.
$$

执行 $r^3$ 次幂后元素不变，其常数项正好是 $s^2\bmod N$。官方 solver 的核心构造如下：

```sage
n, m = 15, 3
r = next_prime(N)
K.<zK> = GF(r).extension(n)
E = K.subfield(m, 'zE')
zE = E.gens()[0]

e0 = int(K(zE).polynomial()[0])
s = randint(1, N)
s2 = s^2 % N

element = K(inverse_mod(e0, r) * s2 * zE) - zK
coeffs = element.polynomial().coefficients(sparse=False)
message = sum(int(a) * r^i
              for i, a in enumerate(coeffs[::-1]))
```

将 `message` 交给服务器签名并取得根 `x`。若恰好得到 $\pm s$，本次没有非平凡因子，需要重新连接再选一个 $s$；否则直接分解：

```sage
p = gcd(s - x, N)
q = N // p
```

恢复 $p,q$ 后，复用题目中的 `sign` 函数为随机认证消息计算合法 $(x,u)$，得到：

```text
DUCTF{lets_us3_a_pr0p3r_h4sh_function_n3xt_t1me...}
```

这个哈希还有更直接的线性弱点：在 $\mathbb F_r$ 上满足
$H(M,u)=H(M,0)+uH(0,1)$。因此也可令 $x=1$，解出使 $H(M,u)=1$ 的 $u$，直接构造通过验证的签名。它进一步说明问题根源在哈希，而不只是 oracle 的选择消息接口。

## 方法总结

Rabin 签名必须先把消息安全地映射到不可控的二次剩余；若攻击者能让签名者对已知平方求根，非平凡平方根就会分解 $N$。本题的扩域哈希既在子域上存在 Frobenius 固定点，又保持线性，无法充当密码学哈希。看到“自定义有限域哈希 + 求模平方根签名”时，应检查固定点、线性、碰撞以及能否控制输出为已知平方。
