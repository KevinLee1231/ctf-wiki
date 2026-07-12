# INTERCEPT

## 题目简述

题目实现了一个基于 Salimi IBE 思路的身份基加密服务。服务端 `INTERCEPT.py` 中 KGC 生成 $N=(zp+1)(2q+1)$，主密钥为 $s_1,s_2$，每个用户注册时会得到私钥 $d=(y_1s_1+y_2s_2)^{-1}\bmod zq$ 以及公开的 $h_1,h_2,e$。题目允许查看 KGC 公参、注册有限个用户、截获一个随机用户的密文，并在提交正确明文后给 flag。漏洞点是多个合法用户私钥之间存在代数关系，足以恢复隐藏阶 $zq$，分解 $N$，再计算等价主密钥，从而解任意用户密文。

## 解题过程

本题是个开放题，有多种解题思路和解题方法，出题人提供一种做法供大家参考，如下文所示：

最近，在*A New Efficient Identity-Based Encryption without Pairing, Salimi* 中提出了一种基于 RSA 模 TDDH 假设的新 IBE。 虽然这个方案是不安全的。 特别是，我们表明，给定 IBE 方案中的多个私钥，可以计算出一对等效的主密钥并完全破坏 KGC。

参考上面提到的论文，我们可以发现，当遵守他们提出的要求的对手可以访问`n`个私钥时，Salimi 的 IBE 方案是不安全的。 也就是说，他们的计划可以完全被打破。

我们有 
$$
d_i = (y_{i_1}s_1 + y_{i_2}s_2)^{-1}\ mod\ zq\\
h_{i_1}\equiv y_{i_1}p\ mod\ zq\\
h_{i_2}\equiv y_{i_2}p\ \ mod\ zq\\
d_j = (y_{j_1}s_1 + y_{j_2}s_2)^{-1}\ mod\ zq\\
h_{j_1}\equiv y_{j_1}p\ mod\ zq\\
h_{j_2}\equiv y_{j_2}p\ mod\ zq\\
$$
我们设置了一组变量 $a_{k_1},a_{k_2}$：
$$
\begin{aligned} a_{k_1}&=d_ih_{i_1}-d_jh_{j_1}\\&= (y_{i_1}p(y_{j_1}s_1 + y_{j_2}s_2)-y_{j_1}p(y_{i_1}s_1 + y_{i_2}s_2))/(y_{i_1}s_1 + y_{i_2}s_2)(y_{j_1}s_1 + y_{j_2}s_2)\\&=ps_2(y_{i_1}y_{j_2}-y_{j_1}y_{i_2})/(y_{i_1}s_1 + y_{i_2}s_2)(y_{j_1}s_1 + y_{j_2}s_2)
\end{aligned}
$$
同样,
$$
\begin{aligned}a_{k_2}&=d_ih_{i_2}-d_jh_{j_2}\\
&=ps_1(y_{i_2}y_{j_1}-y_{j_2}y_{i_1})/(y_{i_1}s_1 + y_{i_2}s_2)(y_{j_1}s_1 + y_{j_2}s_2)
\end{aligned}
$$
很容易观察到
$$
\frac{a_{k_1} }{a_{k_2} } \equiv -\frac{s_2}{s_1}\ mod\ zq
$$
所以我们可以知道：
$$
\frac{a_{k_1} }{a_{k_2} }\equiv \frac{a_{k'_1} }{a_{k'_2} }\ mod\ zq\\
a_{k_1}a_{k'_2}-a_{k'_1}a_{k_2}\equiv 0\ mod\ zq
$$
通过计算 $(a_{k_1}a_{k'_2}-a_{k'_1}a_{k_2})$ in $\mathbb{Z}$, 我们收到隐藏阶 $zq$ 的倍数，不等于 0 的概率很高。

我们知道 $N/2zq=p+p/2q+1/z+1/2zq\in [p, p+1]$ since $N=(zp+1)(2q+1)=2zqp+zp+2q+1$, 这意味着我们可以求解一个方程来分解 N. 然后使用任何 $(h_{i_1},h_{i_2})$我们能够得到 $(y_{i_1},y_{i_2})$, 通过一些计算我们可以得到 $s'_1\equiv s_1,s'_2\equiv s_2\ mod\ zq$.

一旦我们得到这些秘密参数，我们就可以为任意身份计算等价私钥，进而解开被截获用户的密文。

flag: `WMCTF{cracking_such_a_toy_system_is_so_easy!}`

EXP：

```python
from Crypto.Util.number import *
from gmpy2 import *
from random import randint
import hashlib
from string import digits, ascii_letters
from pwn import *
context.log_level = 'debug' 

def proof_of_work(suffix,digest):
    table = digits + ascii_letters
    for i in table:
        for j in table:
            for k in table:
                for l in table:
                        guess = i+j+k+l+suffix
                        if hashlib.sha256(guess.encode()).hexdigest() == digest:
                            return (i+j+k+l)


class user:
    def __init__(self, params):
        self.e, self.d, self.h1, self.h2 = params

def attack():
    '''
    we register `limit_num` users
        (only with their private keys `d_i` and digest values `H1_i`, `h2_i`)
       with using the public params
    to break the KGC
        (factor N = (zp + 1)(2q + 1) and get equivalent master keys `s1`, `s2` in Z_zq)
    '''
    
    sh = remote('0.0.0.0', 12345)
    
    temp = sh.recvline(keepends=False).decode().split(' ')
    suffix, digest = temp[0][-17:-1], temp[-1]
    sh.sendline(proof_of_work(suffix, digest))
    
    sh.sendline('P')
    sh.recvuntil(b'the KGC: ')
    temp = sh.recvline(keepends=False).decode().split(', ')
    N = int(temp[0])
    
    sh.sendline('I')
    sh.recvuntil(b'message ')
    temp = sh.recvline(keepends=False).decode().split(' sent by ')
    username = temp[-1][:-3]
    temp = temp[0].split(', ')
    c1, c2 = int(temp[0][1:]), int(temp[1][:-1])
    
    uu = []
    limit_num = 5
    while len(uu) < limit_num:
        random_username = str(getRandomNBitInteger(20))
        sh.sendline('R')
        sh.recvuntil(b'Input your name: ')
        sh.sendline(random_username)
        temp = sh.recvline()
        if b'Registration Not Allowed!' in temp:
            continue
        sh.recvuntil(b' = ')
        temp = sh.recvline(keepends=False).decode().split(', ')
        e, d, h1, h2 = int(temp[0][1:]), int(temp[1]), int(temp[2]), int(temp[3][:-1])
        uu.append((e, d, h1, h2))
        
    u = [user(uu[i]) for i in range(limit_num)]
    ab = []
    
    def calc(u1, u2):
        a = u1.d * u1.h1 - u2.d * u2.h1
        b = u1.d * u1.h2 - u2.d * u2.h2
        return (a, b)
    
    for i in range(limit_num):
        for j in range(i + 1, limit_num):
            ab.append(calc(u[i], u[j]))

    target = ab[0][0] * ab[1][1] - ab[0][1] * ab[1][0]
    
    for i in range(limit_num):
        for j in range(i + 1, limit_num):
            a1, b1 = ab[i]
            a2, b2 = ab[j]
            target = gcd(a1 * b2 - a2 * b1, target)
    
    app = N // target // 2
    qq, zz = 1, 1
    for pp in range(app, app + 2):
        try:
            aa, bb, cc = 2, 1 + 2 * pp * target - N, pp * target
            qq = (-bb + isqrt(bb ** 2 - 4 * aa * cc)) // (2 * aa)
            if target % qq > 0:
                qq = (-bb - isqrt(bb ** 2 - 4 * aa * cc)) // (2 * aa)
            zz = target // qq
            NN = (zz * pp + 1) * (2 * qq + 1)
            if NN == N:
                break
        except:
            continue
        
    s1_, s2_ = 1, 1
    for i in range(limit_num):
        for j in range(i + 1, limit_num):
            try:
                y11, y12 = u[i].h1 * invert(pp, zz * qq) % (zz * qq), u[i].h2 * invert(pp, zz * qq) % (zz * qq)
                y21, y22 = u[j].h1 * invert(pp, zz * qq) % (zz * qq), u[j].h2 * invert(pp, zz * qq) % (zz * qq)
                ys_1 = invert(u[i].d, zz * qq)
                ys_2 = invert(u[j].d, zz * qq)
                s2_ = invert(y21 * y12 - y11 * y22, zz * qq) * (y21 * ys_1 - y11 * ys_2) % (zz * qq)
                s1_ = invert(y11, zz * qq) * (ys_1 - y12 * s2_) % (zz * qq)
                break
            except:
                pass
               
    print("[+]The KGC is broken.")
    print("[+]Parameters are as follows:")
    print("N = {}\np = {}\nq = {}\nz = {}\ns1_ = {}\ns2_ = {}".format(N, pp, qq, zz, s1_, s2_))
    h1 = int(hashlib.sha256(username.encode()).hexdigest(), 16) 
    h2 = int(hashlib.sha512(username.encode()).hexdigest(), 16) 
    y1 = invert(pp, zz * qq) * h1 % (zz * qq)
    y2 = invert(pp, zz * qq) * h2 % (zz * qq)
    dd = invert(y1 * s1_ + y2 * s2_, zz * qq)
    F = pow(c2, dd, N)
    h3 = int(hashlib.md5(str(F).encode()).hexdigest(), 16)
    m = hex(h3 ^ c1)[2:]
    
    sh.sendline('G')
    sh.sendline(m)
    
    print(sh.recvline())


if __name__ == "__main__":
    attack()
    
```

## 方法总结

本题核心是 IBE 主密钥恢复。注册接口泄露多个用户的 $d,h_1,h_2$ 后，可以构造不同用户对之间的 $a_{k_1},a_{k_2}$，利用比值恒等关系得到隐藏阶 $zq$ 的倍数；对多组结果取 GCD 后恢复 $zq$。

识别信号是：KGC 公参给出 $N,g_1,g_2,g_3$；注册会返回私钥 $d$；用户数量虽有限但足够多；加密目标是随机身份，普通注册该身份不可行，因此需要破 KGC 而不是只攻击某个用户。

复现时先注册约 5 个随机用户名收集私钥信息，计算所有用户对的差分并取 GCD 得到 `target=zq`。利用 $N=(zp+1)(2q+1)$ 和 $N/(2zq)\in[p,p+1]$ 枚举/解方程恢复 $p,q,z$，再反推出等价 $s_1,s_2$，最后为被截获身份计算 $d$ 解密并提交明文。
