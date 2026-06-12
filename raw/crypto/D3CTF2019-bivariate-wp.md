# Bivariate

## 题目简述

题目给出 RSA 素数 $p$ 的中间位信息，可写成 $p = 2^{924}x + y + p_0$，其中 $x,y$ 都是小根。于是得到二元线性同余式，可用 Coppersmith 小根方法恢复 $p$。官方思路参考 “Coppersmith in the Wild” 的低维格构造，用更少的 helpful polynomials 换取速度，LLL 后通过 resultant/GCD 提取两个变量的小根。

题目资料地址：https://gist.github.com/LurkNoi/510357aee9f2f86d91847b82ae07ae9c

题目脚本中参数为 `nbit=2048`、`e=65537`，未知高位和低位长度分别是 `l1=100`、`l2=100`。泄露值写作 `p0 = p & (2**(nbit//2-l2)-2**l1)`，因此可建模为 `pol = 2**924*x + y + p0`，两个变量的搜索界都是 `2**100`。这比单纯“泄露中间位”更具体，也解释了正文中二元小根边界的来源。

## 解题过程

给出 RSA 其中一素数的中间位, 可以得到一个二元一次方程, 典型的 coppersmith method.

对于求解一个多元线性同余式的 small roots, 最先由 Herrmann 和 May 在 Asiacrypt 2008 提出具体解法.

为了最大化可解根的上界, 他们选择了尽可能多的 *helpful* 多项式, 这也意味着高阶的格基.

但是在解题时, 我们对速度更感兴趣.

题目描述中藏有 hint, 与维基中 *Coppersmith method* 的定义对比，可以发现多了 “Coppersmith in the Wild”.

根据论文的实现细节，我们可以使用多项式集合的系数向量构造一个低维的格基  
  
$$G := \{y^h x^i f^j N^l \mid j + l = 1,\ 0 \leq h + i + j \leq k\}$$  
  
并在更短的时间内解决问题. (以上渣翻)

两个月前出的题, 在收到选手的 wp 后, 才发现 11 月上旬 github 上有相关的代码实现了该方案. 另外, 0ops的师傅的三变量解法也很好, 感谢.

```python
PR = PolynomialRing(Zmod(N), names='x,y')
x, y = PR.gens()
pol = 2**924 * x + y + p0

def bivariate(pol, XX, YY, kk=3):
    N = pol.parent().characteristic()

    f = pol.change_ring(ZZ)
    PR,(x,y) = f.parent().objgens()

    idx = [ (k-i, i) for k in range(kk+1) for i in range(k+1) ]
    monomials = map(lambda t: PR( x**t[0]*y**t[1] ), idx)
    # collect the shift-polynomials
    g = []
    for h,i in idx:
        if h == 0:
            g.append( y**h * x**i * N )
        else:
            g.append( y**(h-1) * x**i * f )

    # construct lattice basis
    M = Matrix(ZZ, len(g))
    for row in range( M.nrows() ):
        for col in range( M.ncols() ):
            h,i = idx[col]
            M[row,col] = g[row][h,i] * XX**h * YY**i

    # LLL
    B = M.LLL()

    PX = PolynomialRing(ZZ, 'xs')
    xs = PX.gen()
    PY = PolynomialRing(ZZ, 'ys')
    ys = PY.gen()

    # Transform LLL-reduced vectors to polynomials
    H = [ ( i, PR(0) ) for i in range( B.nrows() ) ]
    H = dict(H)
    for i in range( B.nrows() ):
        for j in range( B.ncols() ):
            H[i] += PR( (monomials[j]*B[i,j]) / monomials[j](XX, YY) )

    # Find the root
    poly1 = H[0].resultant(H[1], y).subs(x=xs)
    poly2 = H[0].resultant(H[2], y).subs(x=xs)
    poly = gcd(poly1, poly2)
    x_root = poly.roots()[0][0]

    poly1 = H[0].resultant(H[1], x).subs(y=ys)
    poly2 = H[0].resultant(H[2], x).subs(y=ys)
    poly = gcd(poly1, poly2)
    y_root = poly.roots()[0][0]

    return x_root, y_root

x, y = bivariate(pol, 2**100, 2**100)
p = 2**924 * x + y + p0
q = N//p
assert p*q==N
```

## 方法总结

- 核心技巧：把已知中间位的 RSA 素数恢复建模为二元小根问题，用低维 Coppersmith 格快速求出未知高/低位。
- 识别信号：RSA 题泄露素数连续中间位，未知部分可拆成两个有界变量时，应考虑 bivariate Coppersmith，而不是只用一元高位/低位攻击。
- 复用要点：格基维度和多项式集合会直接影响速度；LLL 得到多个代数关系后，用 resultant 消去变量，再用 gcd 稳定提取根。

