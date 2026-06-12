# d3qcg

## 题目简述

题目基于二次同余生成器（QCG）的状态递推，只泄露后续状态的高位，需要从高位 hint 中恢复未知低位和 secret。预期参考论文 https://www.iacr.org/archive/pkc2012/72930609/72930609.pdf ，论文研究的是数论型非线性伪随机数生成器：状态按 $v_{i+1}=F(v_i)\bmod N$ 迭代，每轮只输出状态的部分高位；攻击者利用 Coppersmith 小根方法和 LLL 构造格，从已知高位恢复未知低位或初始 seed。对本题来说，它对应到二元小根：已知 $s_2,s_3$ 的高位，低位分别记为 $x,y$，再由二次递推关系建立模 $p$ 的多项式方程。

论文中给出的界大概是：设未知部分 $x<p^\delta$，只要 $\delta < \frac{1}{6}\cdot\frac{2m+1}{m}$，就能利用所述格子规约恢复。不过题目参数没有完全压到论文界内，实际仍可以用常见 Coppersmith 轮子或高位爆破化归完成。

## 解题过程

核心方程来自 QCG 递推：

$$
s_3 \equiv a s_2^2+c \pmod p
$$

设已知高位为 $h_2,h_3$，未知低位上界为 $2^k$，则有

$$
s_2=h_2\cdot 2^k+x,\quad s_3=h_3\cdot 2^k+y
$$

代入后构造二元多项式

$$
f(x,y)=y+h_3\cdot 2^k-a(x+h_2\cdot 2^k)^2-c
$$

对 $f(x,y)\equiv 0\pmod p$ 使用二元 Coppersmith。论文里的造格方式是在有理数域构造目标短向量 $s_0=(1,x_1/X_1,\dots,(x_1/X_1)^{\alpha_1},\dots,(x_n/X_n)^{\alpha_n},0,\dots,0)$，LLL 找到短向量后，再把每一项乘回对应上界来恢复小根。

```python
from Crypto.Util.number import *
import hashlib

def lattice_attack_presu(PR,pol,mm,N,X,Y):  #论文中的格子构造方法

    x,y = PR.gens()
    d = pol.degree()
    polys = Sequence([], pol.parent())
    count = 0
    N_count = []
    for ii in range(1,mm+1):
        for jj in range(0,d*(mm-ii)+1):
            poly = x^jj*f^ii
            N_count.append(ii)
            polys.append(poly)
            count +=1
    B, temp = polys.coefficient_matrix()
    monomials = []
    #polys = sorted(polys)
    for poly in polys:
        monomials+=poly.monomials()
    monomials = sorted(set(monomials))
    #print(monomials)
    num_of_mon=len(monomials)
    dim = num_of_mon+count
    M = matrix(QQ,dim,dim)
    for ii in range(0,num_of_mon):
        M[ii,ii] = 1/(monomials[ii](X,Y))
        #print(M[ii,ii])
    for ii in range(num_of_mon,dim):
        M[ii,ii] = N^N_count[ii-num_of_mon]
        for jj in range(0,num_of_mon):
            M[jj,ii] = B[ii-num_of_mon][num_of_mon-jj-1]
    M = M.LLL()
    for i in range(dim):
        if(M[i][0]==1):
            y = M[i][1]*X
            x = M[i][2]*Y
            print(f"maybe x={x},y = {y}")
            return(x,y)

enc = 6176615302812247165125832378994890837952704874849571780971393318502417187945089718911116370840334873574762045429920150244413817389304969294624001945527125
k = 146
(X,Y) = (2**k,2**k)
paramenter = {'a':
35915186802907199435961371907963662963744845363823800618522370646479694425813919
67815457547858969187198898670115651116598727939742165753798804458359397101, 'c':
69968247529439946318025159211253825200449170951720092200008137186174413557674474
28067985103926211738826304567400243131010272198095205381950589038817395833, 'p':
73865371852403464598577153818355014195330884659847778612689518914820722498225262
23542514664598394978163933836402581547418821954407062640385756448408431347}
a = paramenter['a']
c = paramenter['c']
p = paramenter['p']
hint =
[6752358399910239128664664867482701208988865057671533314741736291970634913733757
0430286202361838682309142789833,
70007105679729967877791601360700732661124470473944792680253826569739619391572400
148455527621676313801799318422]
(h2,h3) = hint
PR.<x,y>  = PolynomialRing(QQ)
f = (y+h3*pow(2,k))-(a*(x+h2*pow(2,k))^2+c)
(l2,l3) = lattice_attack_presu(PR,f,3,p,X,Y)
s2 = h2*2**k+l2
s3 = h3*2**k+l3
print(s2,s3)
secret = mod((s2-c)*inverse_mod(a,p)%p,p).sqrt()
print(f"secret = {secret}")
flag  = long_to_bytes(bytes_to_long(hashlib.sha512(b'%d'%
(secret)).digest())^^enc)
print(flag)
#b'Here_is_ur_flag!:)d3ctf{th3_c0oppbpbpbp3rsM1th_i5_s0_1ntr35ting}'
```

## 方法总结

- 核心技巧：把相邻 QCG 状态的高位泄露写成 $s_i=h_i2^k+\text{low}_i$，再用递推关系构造二元小根方程。
- 外链论文的作用：它提供了“非线性 PRNG + 高位泄露 + Coppersmith/LLL”的通用攻击框架，本题只是把该框架落到二次递推和两个未知低位上。
- 复用要点：如果严格小根界不够，优先考虑爆破少量高位/低位把参数压进现有 Coppersmith 轮子的可解范围。
