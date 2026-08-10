# RSA 大冒险 2

## 题目简述

题目由三个连续的 RSA 关卡组成，每关使用新的动态参数：

1. 私钥指数 $d$ 被限制在远小于 $N^{1/4}$ 的范围内；
2. 两个素因子距离很近，同时公钥指数 $e$ 与 $\varphi(N)$ 的最大公因数为 $2$；
3. 泄露了素因子 $p$ 的高位，但未知低位恰好卡在常规 Coppersmith 参数的边界附近。

每一关都要从服务端取得当次实例的 $N,e,c$ 及相应泄露，恢复关卡秘密后提交，最终由服务端返回 flag。官方题解只保留了生成逻辑和攻击方法，没有保存动态实例参数或最终服务器 flag，因此下面给出的是可直接填入当次参数的完整求解过程，而不虚构固定 flag。

## 解题过程

### Challenge 1：Wiener 攻击

密钥生成器令

$$
d<\frac{N^{1/4}}{3},
$$

且 $p,q$ 大小接近。这正好落入 Wiener 攻击的适用范围。由

$$
ed-k\varphi(N)=1
$$

可知 $k/d$ 是 $e/N$ 的某个连分数收敛分数。依次枚举收敛分数 $(k,d)$，若 $k\ne0$ 且 $k\mid(ed-1)$，便可得到候选

$$
\varphi(N)=\frac{ed-1}{k}.
$$

再由

$$
p+q=N-\varphi(N)+1,
$$

检查二次方程 $x^2-(p+q)x+N=0$ 的判别式是否为完全平方数。下面的脚本同时完成连分数恢复、因子验证和解密：

```python
from math import isqrt
from Crypto.Util.number import long_to_bytes


def continued_fraction(numerator, denominator):
    result = []
    while denominator:
        quotient, remainder = divmod(numerator, denominator)
        result.append(quotient)
        numerator, denominator = denominator, remainder
    return result


def convergents(coefficients):
    old_num, num = 0, 1
    old_den, den = 1, 0
    for coefficient in coefficients:
        new_num = coefficient * num + old_num
        new_den = coefficient * den + old_den
        yield new_num, new_den
        old_num, num = num, new_num
        old_den, den = den, new_den


def wiener_attack(N, e):
    for k, d in convergents(continued_fraction(e, N)):
        if k == 0 or (e * d - 1) % k:
            continue

        phi = (e * d - 1) // k
        factor_sum = N - phi + 1
        discriminant = factor_sum * factor_sum - 4 * N
        if discriminant < 0:
            continue

        root = isqrt(discriminant)
        if root * root != discriminant:
            continue
        if (factor_sum + root) % 2:
            continue

        p = (factor_sum + root) // 2
        q = (factor_sum - root) // 2
        if p * q == N:
            return p, q, d
    raise ValueError("Wiener attack failed")


N = int(input("N = "), 0)
e = int(input("e = "), 0)
c = int(input("c = "), 0)

p, q, d = wiener_attack(N, e)
assert p * q == N
message = pow(c, d, N)
print(long_to_bytes(message))
```

### Challenge 2：Fermat 分解与模平方根

这一关按

$$
q=\operatorname{next\_prime}(p+r),\qquad r<2^{256}
$$

生成两个约 512 位素数，所以 $p$ 与 $q$ 相对于自身规模非常接近。写成

$$
N=pq=a^2-b^2=(a-b)(a+b)
$$

后，从 $a=\lceil\sqrt N\rceil$ 开始寻找使 $a^2-N$ 为完全平方数的值，即可用 Fermat 方法快速得到 $p=a-b,q=a+b$。

另一个陷阱是 $\gcd(e,\varphi(N))=2$，所以 $e$ 在模 $\varphi(N)$ 下没有逆元。令

$$
t=\gcd(e,\varphi(N))=2,\qquad e'=e/t,
$$

此时 $e'$ 可逆。若 $d'=(e')^{-1}\bmod\varphi(N)$，则

$$
c^{d'}\equiv m^{ed'}\equiv m^t\equiv m^2\pmod N.
$$

分别在模 $p$、模 $q$ 下开平方，再用中国剩余定理组合，会得到至多四个候选明文。通过可打印字符和题目规定的答案格式即可选出关卡秘密：

```python
from math import gcd, isqrt
from Crypto.Util.number import long_to_bytes
from sympy import sqrt_mod
from sympy.ntheory.modular import crt


def fermat_factor(N):
    a = isqrt(N)
    if a * a < N:
        a += 1

    while True:
        difference = a * a - N
        b = isqrt(difference)
        if b * b == difference:
            p = a - b
            q = a + b
            if p * q == N:
                return p, q
        a += 1


N = int(input("N = "), 0)
e = int(input("e = "), 0)
c = int(input("c = "), 0)

p, q = fermat_factor(N)
phi = (p - 1) * (q - 1)
t = gcd(e, phi)
assert t == 2

reduced_e = e // t
reduced_d = pow(reduced_e, -1, phi)
message_squared = pow(c, reduced_d, N)

roots_p = sqrt_mod(message_squared, p, all_roots=True)
roots_q = sqrt_mod(message_squared, q, all_roots=True)

for root_p in roots_p:
    for root_q in roots_q:
        candidate = int(crt([p, q], [root_p, root_q])[0])
        plaintext = long_to_bytes(candidate)
        if all(byte in b"\t\n\r" or 0x20 <= byte < 0x7F for byte in plaintext):
            print(plaintext)
```

### Challenge 3：高位泄露与 Coppersmith 小根

第三关泄露

$$
\text{leak}=p\mathbin{\gg}253,
$$

所以可写成

$$
p=(\text{leak}\ll253)+x,\qquad 0\le x<2^{253}.
$$

直接用 `small_roots(X=2^253, beta=0.4)` 位于参数边界，原题实例难以稳定求根。官方解法再枚举未知部分最高的 5 位，把真正交给格规约的未知量缩小到 248 位：

$$
p=((\text{leak}\ll5)+t)\ll248+x,
\quad 0\le t<2^5,
\quad 0\le x<2^{248}.
$$

同时把 `epsilon` 调低到 `0.01`，增加 Coppersmith 构造格的维度，以换取更大的可求根边界。找到小根后，候选多项式值就是 $p$；验证其整除 $N$，即可恢复私钥并解密：

```python
from Crypto.Util.number import long_to_bytes
from sage.all import PolynomialRing, Zmod, gcd

N = int(input("N = "), 0)
e = int(input("e = "), 0)
c = int(input("c = "), 0)
leak = int(input("leak = "), 0)

shift_bits = 253
guessed_bits = 5
unknown_bits = shift_bits - guessed_bits

ring = PolynomialRing(Zmod(N), "x")
x = ring.gen()

p = None
for guess in range(1 << guessed_bits):
    known_high = ((leak << guessed_bits) + guess) << unknown_bits
    polynomial = known_high + x
    roots = polynomial.small_roots(
        X=1 << unknown_bits,
        beta=0.4,
        epsilon=0.01,
    )
    for root in roots:
        candidate = int(polynomial(root))
        factor = int(gcd(candidate, N))
        if 1 < factor < N and N % factor == 0:
            p = factor
            break
    if p is not None:
        break

if p is None:
    raise ValueError("no factor recovered")

q = N // p
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
plaintext = long_to_bytes(pow(c, d, N))
print(plaintext)
```

[App1e_Tree 的 HGAME2023 Crypto 复现](https://www.cnblogs.com/App1eTree/p/2023hgame.html)记录的第三关明文为：

```text
now_you_know_how_to_use_coppersmith
```

这个字符串只是第三关需要提交的秘密，不是比赛最终 flag；最终 flag 仍由当时的在线服务在三关全部通过后返回。

## 方法总结

三关分别对应 RSA 中三类常见的参数失误：私钥指数过小、素因子过近且指数不可逆、以及素因子高位泄露。判断攻击方向时应直接对照生成器中的不等式和最大公因数，而不能看到 RSA 就只尝试通用分解。尤其在 Coppersmith 临界实例中，`X`、`beta` 与 `epsilon` 决定格的规模和可恢复根界；先穷举少量相邻高位、再用更高维格处理剩余未知位，通常比盲目延长默认参数的运行时间更有效。
