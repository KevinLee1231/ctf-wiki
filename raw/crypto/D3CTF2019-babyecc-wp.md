# babyecc

## 题目简述

题目考察模合数上的椭圆曲线离散对数。曲线参数 $A$ 需要借 Carmichael 函数性质确定；模 $N=pq$ 的曲线点运算可分解到模 $p$ 和模 $q$ 上分别求解，再用中国剩余定理合并离散对数结果。解题流程是先构造曲线参数 $A,B$，在 $E(\mathbb{F}_p)$ 和 $E(\mathbb{F}_q)$ 上分别求 $Q=dP$ 的 $d_1,d_2$，最后 CRT 还原 $d$ 并转成 flag。

## 解题过程

出题思路来自于 `CryptoCTF 2019` 的 `Super Natural` ，但是当时出题人参数设置失误，导致该题无解。

在本题中，求曲线的参数 $A$，需要用到 `Carmichael theory` （卡米歇尔定理）

`Carmichael theory` （卡米歇尔定理）：对满足 $\gcd(a, n) = 1$ 的所有 $a$，使得 $a^m \equiv 1 \pmod{n}$ 同时成立的最小正整数 $m$，称为 $n$ 的卡米歇尔函数，记为 $\lambda(n)$。

本题中有 $\lambda(n) = \frac{1}{2}\varphi(n) = 2^{\alpha-2},\quad n = 2^{\alpha},\ \alpha > 2$

第二个则考察中国剩余定理

在模 $n$ 的曲线上能够分解到模 $p$，模 $q$ 的曲线上，并且能够使用中国剩余定理还原的简单证明如下：

> 对于 $p, q$ 是素数，$n = p \times q$，则 $a \equiv b \pmod{n}$ 等价于  
> $$\begin{cases}
> a \equiv b \pmod{p} \\
> a \equiv b \pmod{q}
> \end{cases}$$

上述式子在椭圆曲线上也成立

`exp` 如下

```python
mid_a = euler_phi(pow(2, 253)) / 2
useless_num1 = 84095692866856349150465790161000714096047844577928036285412413565748251721
A = mid_a + useless_num1
N = 45260503363096543257148754436078556651964647703211673455989123897551066957489
p = 136974486394291891696342702324169727113
q = 330430173928965171697344693604119928553

P = (44159955648066599253108832100718688457814511348998606527321393400875787217987,
     41184996991123419479625482964987363317909362431622777407043171585119451045333)
Q = (6856607779216667472822134501915718711944054464462866581688216679749429584974,
     37306843623514161736361170354186251255656545843342522443684235257196063601639)
x_P, y_P = P
x_Q, y_Q = Q
B = (y_P^2 - x_P^3 - A * x_P) % N

F = IntegerModRing(N)
F1 = FiniteField(p)
F2 = FiniteField(q)
E = EllipticCurve(F, [A, B])
E1 = EllipticCurve(F1,[A,B])
E2 = EllipticCurve(F2,[A,B])

print(E1.order().factor())
print(E2.order().factor())

P = E(P)
Q = E(Q)
P1 = E1(P)
P2 = E2(P)
Q1 = E1(Q)
Q2 = E2(Q)

print("-" * 50)
print(P1)
print(Q1)
d1 = discrete_log(Q1, P1, P1.order(), operation="+")
print(d1 * P1)
print("d1 =", d1)
print(P2)
print(Q2)
d2 = discrete_log(Q2, P2, P2.order(), operation="+")
print(d2 * P2)
print("d2 =", d2)
print("-" * 50)

d = crt([d1, d2], [P1.order(), P2.order()])
print(d)
assert d * P == Q

flag = "d3ctf{" + hex(d).decode("hex") + "}"
print(flag)
```

## 方法总结

- 核心技巧：把合数模椭圆曲线问题拆成两个素域曲线上的离散对数，再用 CRT 合并指数。
- 识别信号：椭圆曲线定义在 `Zmod(N)` 且已知或可分解 `N=pq`，同时给出 $P,Q$ 满足 $Q=dP$ 时，应检查是否能降到 $E(\mathbb{F}_p)$、$E(\mathbb{F}_q)$。
- 复用要点：Carmichael 函数用于确定曲线参数时要注意 $2^\alpha$ 的特殊形式；两个子群阶不一定互素，CRT 前需要确认离散对数解与阶的兼容性。

