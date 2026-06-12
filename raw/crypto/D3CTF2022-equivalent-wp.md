# equivalent

## 题目简述

考点是 equivalent key attack。题目使用类似 knapsack/subset-sum 的公钥结构，公开向量 $\vec a$ 由私钥奇数序列 $\vec s$、乘子 $e$ 和模数 $p$ 生成。攻击目标不是恢复出题时的原始私钥，而是构造一组“等效私钥” $(\vec s,e,p)$，只要它满足同样的解密条件，就能解出密文。

论文《Equivalent Secret Key Attack against Knapsack PKC based on Subset Sum Decision Problem》的关键结论是：对基于子集和判定问题的 knapsack 公钥密码，公开向量所在的正交格会泄露包含等效私钥的低维结构；攻击者可在该低维格中寻找满足正数、奇偶性、模数大于权重和等条件的另一组私钥参数，从而完成解密。

## 解题过程

观察解密流程可以得出等效密钥需要满足以下条件:

1. $a_i = e s_i \bmod p$

2. e, p 互素且 $p > \sum_i s_i$

3. $s_i$ 为正奇数

设 $\vec{a} = e \vec{s} + p \vec{k}$

利用 orthogonal lattice 可以求出包含向量 $\vec{s}, \vec{k}$ 的格 $\mathbb{L}_1$ , 证明参见

Equivalent key attack against a public-key cryptosystem based on subset sum problem

记 $\mathbb{L}_1$ 的基底为 $\vec{u_1}, \vec{u_2}$, 且

$$
\vec{s}=x_1\vec{u_1}+x_2\vec{u_2}
$$

$$
\vec{k}=y_1\vec{u_1}+y_2\vec{u_2}
$$

$$
\vec{a}=z_1\vec{u_1}+z_2\vec{u_2}
$$

则由条件3可以确定 $x_1, x_2$ 的奇偶性

条件1等价于

$$
\begin{pmatrix}e\\p\end{pmatrix}^{T}\cdot\begin{pmatrix}x_1 & x_2\\y_1 & y_2\end{pmatrix}=(z_1\quad z_2)
$$

注意到 $p > e \gg a \gg z_i$, 故 $|\frac{x_1}{x_2}| - |\frac{y_1}{y_2}|$ 非常小

因此可以先随机选取 $x_1, x_2$ 满足条件3, 再确定 $y_1, y_2$ 并解出 $e, p$ 最后检验条件2

完整 exp:

```sage
from collections import namedtuple


PublicKey = namedtuple('PublicKey', ['a'])
SecretKey = namedtuple('SecretKey', ['s', 'e', 'p'])

def bits2bytes(bits):
    num = int(''.join(map(str, bits)), 2)
    b = num.to_bytes((len(bits)+7)//8, 'big')
    return b

def dec(c, sk):
    d = inverse_mod(sk.e, sk.p)
    m = (d * c % sk.p) % 2
    return m

def decrypt(cip, sk):
    msg = bits2bytes([dec(c, sk) for c in cip])
    return msg



exec(open('data.txt', 'r').read())
#pk = ...
#cip = ...

n = len(pk.a)


def orthogonal_lattice(B):

    LB = B.transpose().left_kernel(basis="LLL").basis_matrix()

    return LB

a = vector(ZZ, pk.a)
La = orthogonal_lattice(Matrix(a))

L1 = orthogonal_lattice(La.submatrix(0, 0, n-2, n))
u1, u2 = L1

L1_m = L1.change_ring(Zmod(2))
x1_m, x2_m = L1_m.solve_left(vector(Zmod(2), [1]*n)).change_ring(ZZ)

z = L1.solve_left(a).change_ring(ZZ)


def gen_close(x1, x2):
    cc = list((x1 / x2).continued_fraction())

    if randint(0, 1): # 两种都能出
        cc[-1] -= 1
    else:
        cc = cc[:-1]
        cc[-1] += 1

    cc = continued_fraction(cc).convergents()[-1]

    return cc.numer(), cc.denom()

K = 20 # 太大容易 sum(s) > p

for _ in range(16):
    while True:
        xx1 = randint(-2**K, 2**K)*2 + x1_m
        xx2 = randint(-2**K, 2**K)*2 + x2_m
        ss = xx1*u1 + xx2*u2
        if min(ss) > 0:
            break

    yy1, yy2 = gen_close(xx1, xx2)
    ee, pp = Matrix(ZZ, [
        [xx1, xx2],
        [yy1, yy2]
    ]).solve_left(z)

    if not ee.is_integer() or ee < 0:
        continue

    if not pp.is_integer():
        continue

    if pp < 0:
        yy1, yy2 = -yy1, -yy2
        pp = -pp

    if gcd(ee, pp) != 1:
        continue

    if sum(ss) > pp:
        continue

    sk = SecretKey(ss.list(), ee, pp)
    print(decrypt(cip, sk))
    break
```

## 方法总结

- 核心技巧：先从公开向量 $\vec a$ 构造正交格，再把包含 $\vec s,\vec k$ 的二维格提取出来，避免在原始高维空间里直接搜索。
- 等效私钥的判断条件：$\vec s$ 为正奇数，$e,p$ 互素，且 $p>\sum_i s_i$；满足这些条件即可正常解密，不要求与原始私钥一致。
- 复用要点：knapsack 类题目出现“解密只依赖同余和奇偶性”时，要优先检查能否构造 equivalent key，而不是只尝试还原出题时的真实 secret key。
