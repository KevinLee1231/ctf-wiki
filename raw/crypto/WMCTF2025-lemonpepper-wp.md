# LemonPepper

## 题目简述

题目给出了flavorings类和LemonPepper类，其中flavorings类具有mcg结构，作为LemonPepper类的伪随机数生成器。整个题目给出了Lemon和Pepper两组数据，要求恢复最终flavorings类中的state。

附件 `chal.sage` 的关键参数为 `p=257,l=25`，以及 `q=27658548947,t=30,e1=135,e2=165`。服务端先输出 7 个 `Lemon()` 多项式，再输出 3 个 `Pepper()` 多项式，最后要求输入当前 `flavorings.state` 的 25 个整数。`flavorings.next()` 每次从 $d=1,2,3$ 对应的三个递推值中随机选一个，递推值形如 $\sum_i state_i^d a_i \bmod 257$，因此恢复 `a_i` 后还要处理三选一导致的状态同步问题。

## 解题过程

### Lemon 部分：重根多项式求解

1. 第一个 part，我们调用7次Lemon函数，其在$q^{135}$下返回
$$
f(x)=c\prod_i(x-\text{root}_i)^{e_i} \mod q^{165},e_i∈\{2,3,4\}
$$
我们的目标是恢复$\text{root}_i$集合中包含伪随机数生成器的解，这可以通过求解$f$函数来完成。根据一般意义上的Hensel Lift，对于$f$的一个解$x$，有迭代公式
$$
x_{k+1}=x_k-f(x_k)·(f'(x_1))^{-1} \mod q^{k+1}
$$
而在重根的情况下，存在
$$
f'(x_1)≡0 \mod q,
$$
我们没法直接进行Hensel Lift来获得$q^k$上的解$x$（这也是在SageMath中使用.roots函数会卡住的原因）。要在这个条件下对方程进行求解，我们就要想办法把方程从重根方程变成无重根方程。由于本题的根的重数在$\{2,3,4\}$中选取，因此我们对函数式进行三次求导，就能够将重数为$4$的根保留在多项式中。构建函数$g$如下
$$
g=\gcd(f,f''')
$$
$g$在$p^{165}$上是可解方程。故我们可以恢复所有的重数为$4$的解，恢复后将重数为$4$的项从$f$中删除，即可将函数式的重数选取范围降至$\{2,3\}$。以此类推，即可完成Lemon部分的求解。

### MCG 状态恢复

2. 求解完7个多项式的解，我们可以恢复得到mcg的$945$个生成量。观察mcg的生成函数，有
$$
s_{n+26}=\sum_{i=0}^{25}(s_{n+i})^ea_i \mod p,e∈\{1,2,3\}
$$
使用全线性化处理式子，有
$$
\prod_{e=1}^3(\sum_{i=0}^{25}(s_{n+i})^ea_i-s_{n+26})=0 \mod p
$$
我们共有$m=945$个式子，但存在$\tbinom{n+3}{n}=3275$个单项式，式子数量远小于单项式数量。故采取Arora-Ge使用的技巧增加样本数量：对于任意一个式子，我们对其分别乘以所有度小于$d$的单项式，此时式子数量变成$m\tbinom{n+d}{d}$，单项式数量变成$\tbinom{n+d+3}{n}$。取$d=1$，当样本数量为$m=914$时，式子数量为$23764$，单项式数量为$23750$。而为了构成方程，我们还需要额外的$n$组样本，总共需要$939$个mcg输出，小于我们拥有的输出量。构造对应的矩阵方程，我们可以完整求解得到所有的$a_i$。

### Pepper 部分：p-adic 求解与剪枝

3. 恢复所有的$a_i$后，我们已经拥有了mcg的完整状态，即使如此，mcg的每一次随机数生成都伴随着一次随机的三选一，因此我们仍然需要去跟进mcg的输出。对于第二个part，我们调用三次Pepper函数，其在$q^{165}$下返回
$$
f(x)=c\prod_i(x-\text{root}_i)(x-\text{root}_i-q^{30})·\prod_{\text{root}∈sample(\text{roots},2)}(x-\text{root}-q^{40}) \mod q^{165}
$$
相比part1，本部分生成了一个在低指数下具有重根的式子。即使采用part1中的方法，我们也只能在$q^{30}$上求出对应的解，想要继续抬升，则又会受到导数等于$0$的限制。而$q$的大小为$30$bit，意味着我们无法采用爆破的方式去进行深搜。这样，我们只能采取在p-adic域上定义函数，以牺牲精度为代价求取一定范围上的解。对于本题，我们可以在$q^{135}$上恢复所有不在$sample(\text{roots},2)$中的元素的解。这样，我们也就变相得到了30组mcg生成随机数中的28组，其顺序未知。

4. 由于在part2之前整个mcg类对我们而言已经透明，而求解得到的列表包含30组随机生成数中的28组，我们可以采用剪枝算法来确定生成顺序的唯一解。注意到剪枝过程中可能会出现多解，成功存在一定概率性，成功率大概为70%左右。

```python
from Crypto.Util.number import *
from time import time
from pwn import *

def solvelemon(p, poly, e = 165):
    R.<x> = PolynomialRing(Zmod(p^e))

    poly = R(poly)
    def gcd(x, y):
        while True:
            x, y = y, x % y
            if y == 0:
                return x

    for i in range(3):
        g = poly
        for j in range(3-i):
            g = diff(g)

        for j in gcd(poly,g).roots(multiplicities=False):
            ans = int(j)
            s = []
            for _ in range(165):
                s.append(int(ans % p))
                ans //= p

            if all([_ < 257 for _ in s[30:]]):
                return s[30:]
            poly //= (x-j)^(i+1)

def linearation(c):
    from tqdm import trange

    p, n = 257, 25
    R = PolynomialRing(GF(p),[f'x{i}' for i in range(n)])
    x = [i for i in R.gens()]+[1]

    dic = {}
    m = []
    v = []
    for i in trange(914):
        f = prod(sum(c[i:i+n][j]^d*x[j] for j in range(n))-c[i+n] for d in range(1,4))
        for d1 in range(n+1):
            vr, term = 0, x[d1]
            g = f*term
            coes, mons = g.coefficients(), g.monomials()
            mr = [0 for i in range(23750)]
            for mon,coe in zip(mons,coes):
                if mon == 1:
                    vr = -coe
                    continue
                if mon not in dic.keys():
                    dic[mon] = len(dic)
                mr[dic[mon]] = coe
            m.append(mr)
            v.append(vr)

    v = matrix(v)
    al = matrix(GF(p),m).T.solve_left(v).list()
    a = []
    for i in x[:-1]:
        a.append(al[dic[i]])
    return a

def solvepepper(poly, e = 165):
    R.<x> = PolynomialRing(Zmod(p^e))
    f = R(poly)

    Qpp = Qp(p, prec=e)
    R.<x> = PolynomialRing(Qpp)
    re = R(f).roots()

    nexts = []
    for i in re:
        ans = eval(str(i[0]).split('O')[0][:-3].replace('^','**'))
        if ans.bit_length() > 4000:
            nexts.append(int(ans%p^135)//p^134)
    return nexts[::2]

def guess(s, a, nexts, label):
    global ans
    if nexts == [-1]*25:
        ans = s
        print(1)
    nc = [sum(s[i] ^ d * a[i] for i in range(30)) % 257 for d in range(1, 4)]
    for i in nc:
        if i in nexts:
            nexts1 = deepcopy(nexts)
            nexts1[nexts1.index(i)] = -1
            guess([i]+s, a, nexts1, label)
        else:
            if label < 5:
                guess([i]+s, a, nexts, label+1)


from pwn import *
context.log_level = 'error'

p = 27658548947
re = remote('',)

re.recvuntil(b'Injection!\r\n')
s = []
for i in range(7):
    s += solvelemon(p,eval(re.recvline()))
print(s)
a = linearation(s)[::-1]
s = s[-25:][::-1]

re.recvuntil(b'Ignition!\r\n')
for i in range(3):
    nexts = solvepepper(eval(re.recvline()))
    global ans
    ans = []
    guess(s,a,nexts,0)
    s = ans[:25]

re.recvuntil(b'> ')
re.sendline(str(s)[1:-1].encode())
print(re.recvline())
print(re.recvline())
```

## 方法总结

- 核心技巧：Lemon 阶段用求导和 gcd 分离重根，恢复 MCG 输出；再用 Arora-Ge 线性化恢复 MCG 系数；Pepper 阶段在 p-adic 域牺牲精度求部分根，并用剪枝同步随机输出顺序。
- 识别信号：多项式根有固定重数且普通 Hensel lift 卡在导数为 0 时，应考虑对多项式求导并用 gcd 分层剥离重根。
- 复用要点：Arora-Ge 的关键是用低度单项式乘原方程增加样本数量，使线性化后的方程数超过单项式数；p-adic 解只恢复部分候选时，后续要结合 PRNG 递推和剪枝确定顺序。
