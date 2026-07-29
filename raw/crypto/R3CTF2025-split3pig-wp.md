# split3pig

## 题目简述

题目给出一个非标准 RSA 模数：

$$
N=P\cdot Q^2,
$$

以及两个约 244 位的公开整数 $E_1,E_2$，满足：

$$
E_1\mid P-1,\qquad E_2\mid Q-1.
$$

$N$ 约 2607 位，$Q$ 约 870 位。目标不是解密密文，而是恢复 $Q$，最终 flag 为：

```python
"r3ctf{" + sha256(str(Q).encode()).hexdigest() + "}"
```

公开整除关系先泄露 $Q$ 模 $E_1E_2$ 的剩余类，再用针对未知因子 $Q^2$ 的单变量 Coppersmith 找回高位。

## 解题过程

### 求出 Q 的部分剩余类

由 $P\equiv1\pmod{E_1}$：

$$
N=P Q^2\equiv Q^2\pmod{E_1}.
$$

因此先在 $\mathbb Z/E_1\mathbb Z$ 中求：

$$
r_1^2\equiv N\pmod{E_1}.
$$

`E1` 可分解为多个互异素因子，所以平方根不唯一；题目实例共得到 $2^5=32$ 个候选。另一方面：

$$
Q\equiv1\pmod{E_2}.
$$

对每个平方根 $r_1$ 使用 CRT：

```python
rq = crt([r1, 1], [E1, E2])
```

便得到一个候选：

$$
Q\equiv r_Q\pmod{E_1E_2}.
$$

逐个测试这 32 个候选即可，不必事先判断平方根符号。

### 针对 Q 的平方因子做小根攻击

写成：

$$
Q=kE_1E_2+r_Q.
$$

$E_1E_2$ 约 487 位，而 $Q$ 约 870 位，所以未知的 $k$ 只有约 383 位。构造：

$$
f(X)=(XE_1E_2+r_Q)^2\pmod N.
$$

正确的 $k$ 满足：

$$
f(k)=Q^2\equiv0\pmod{Q^2},
$$

而 $Q^2$ 是 $N$ 的约 $2/3$ 位因子。Sage 中可按官方求解器参数调用：

```python
qbit = 870
K = 2 ** (qbit - E1.nbits() - E2.nbits())

R.<x> = PolynomialRing(Zmod(N))
f = (x * E1 * E2 + ZZ(rq)) ** 2
roots = f.monic().small_roots(
    X=K,
    epsilon=0.05,
    beta=2/3 - 0.1,
)
```

这里 `beta` 描述未知模因子 $Q^2$ 相对 $N$ 的规模。对错误的 $r_Q$ 不会得到有效根；命中后计算：

```python
Q = int(rq) + int(roots[0]) * E1 * E2
assert N % (Q * Q) == 0
```

再检查 $Q$ 为素数、`(Q - 1) % E2 == 0`，即可排除偶然解。

### 生成 flag

题目没有额外的编码层。恢复十进制整数 $Q$ 后，严格按照源码先执行 `str(Q).encode()`，再计算 SHA-256；不要对大整数的二进制字节表示做哈希。

## 方法总结

本题利用了 $\varphi$ 隐藏类问题中的公开大因子：$E_1\mid P-1$ 让 $N\bmod E_1$ 退化成 $Q^2$，$E_2\mid Q-1$ 又固定了第二个剩余类。CRT 之后已知约 487 个低位规模的信息，而未知部分远小于对 $N$ 的大因子 $Q^2$ 可承受的小根界，因而可以直接恢复 $Q$。

可运行的 Sage 参数和 32 个平方根候选的遍历方式见 [tl2cents 的 split3pig 求解器](https://github.com/tl2cents/CTF-Writeups/tree/master/2025/R3CTF/split3pig)；正文已经保留生成多项式、界限和最终哈希格式。
