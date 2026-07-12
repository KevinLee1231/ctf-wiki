# welcome_signer1

## 题目简述

题目服务生成 RSA 密钥，固定消息为 `Welcome_come_to_WMCTF`，并提供三个主要功能：`G` 返回公钥 `n` 和用私钥 `d` 派生 AES key 加密后的 flag；`F` 允许把 RSA 模数 `N` 的某一字节翻转成故障模数 $\hat N$，且只有一次机会；`S` 在指定干扰位置 `j` 下返回故障签名。目标是从故障签名 oracle 中恢复私钥 `d`，再用 `md5(str(d))` 派生 AES-ECB key 解密 flag。

外部论文《Fault Attacks on RSA Public Keys: Left-To-Right Implementations are also Vulnerable》的关键结论是：即使攻击者只能扰动公钥模数而不是私钥或消息，只要能控制故障发生在模幂算法的特定步骤，就可以比较正确签名与故障签名，逐位恢复私钥指数。本文使用的版本从低位恢复 `d`，并要求选择一个便于开方/判定二次剩余的故障模数 $\hat N$。

## 解题过程

参考《Fault Attacks on RSA Public Keys: Left-To-Right Implementations are also Vulnerable》

简要描述：使用从右到左算法进行签名计算。

```python
def Left_to_Right(m,d,N):
    A = 1
    d = d.bits()[::-1]
    n = len(d)
    for i in range(n-1,-1,-1):
        A = A*A % N
        if d[i] == 1:
            A = A * m % N
    return A
```
错误注入模型

```python
def fault_left_to_right_exp(m,d,N,j,N_):
    A = 1
    d = d.bits()[::-1]
    n = len(d)
    for i in range(n-1,-1,-1):
        if i < j:
            #print(A)
            N = N_
        A = A*A % N
        if d[i] == 1:
            A = A * m % N
    return A
```
正确签名
$$
S \equiv m^{\sum_{i=0}^{n-1}2^i\cdot d_i} \pmod N	
$$
错误注入方式为：
$$
A \equiv m^{\sum_{i=j}^{n-1}2^{i-j}\cdot d_i} \mod N
$$
错误签名表达式为：
$$
\hat S \equiv (((A^2\cdot m^{d_{j-1}})^2 \cdot m^{d_{j-2}})^2 \cdots )^2 \cdot m^{d_0} \mod \hat N \\
\equiv A^{2j} \cdot m^{\sum_{i=0}^{j-1}2^i \cdot d_i} \mod \hat N
$$
攻击条件：已知 $N,\hat N,S$，注入位置 $j$（可控）已知，错误模数 $\hat N$ 可分解或为素数。从低位爆破私钥 $d'$，

首先计算
$$
R = \hat S \cdot m^{-d'}
$$
然后验证 $R$ 是否为二次剩余，若是则进行开方，共开 $j$ 次。注意每次开方时需对两个根进行判断，舍弃其中的非二次剩余，从而保证解集始终只有两个。
$$
R = R^{\frac{1}{2^j}}
$$
修正
$$
S' \equiv  R^{2^j} \cdot m^{d'} \mod N
$$
检查是否满足 $S' \equiv S \mod N$

```python
from Crypto.Util.number import *
from Crypto.Cipher import AES
from hashlib import md5
from sympy import isprime
from tqdm import tqdm 
import random
from pwn import *

# context.log_level = 'debug'

def get_sig(j):
    sh.recvuntil("[Q]uit\n")
    sh.sendline("S")
    sh.recvuntil("interfere:")
    sh.sendline(str(j))
    sh.recvuntil(" is ")
    sigs = sh.recvuntil("\n")[:-1]
    return int(sigs)

def decrypt(message,key):
    key = bytes.fromhex(md5(str(key).encode()).hexdigest())
    enc = AES.new(key,mode=AES.MODE_ECB)
    c   = enc.decrypt(message)
    return c



#sh = process(["python","task.py"])
sh = remote("0.0.0.0",9999)
sh.recvuntil("[Q]uit\n")
sh.sendline("G")

sh.recvuntil("| n = ")
n = int(sh.recvuntil("\n")[:-1])
sh.recvuntil("ciphertext = ")
cipher = bytes.fromhex(sh.recvuntil("\n").decode()[:-1])



for _ in range(10000):
    index = random.randint(0,1024)
    temp = random.randint(0,256)
    n_ = n ^ (temp<<index)
    if isprime(n_) and n_ % 4==3:
        print("[+]",n_)
        break
else:
   exit()
sig = get_sig(0)
print("[+]",sig)

print("[+] cipher",cipher)
print("[+] n",n)

sh.recvuntil("[Q]uit\n")
sh.sendline("F")
sh.recvuntil("index:")
sh.sendline(",".join([str(temp),str(index)]))

# poc ps: associate with coppersmith will be more efficient

dd=0

for j in tqdm(range(1,300)):
    sig_ = get_sig(j)
    #print(j)
    for i in range(2):
        d_ = (i<<j-1) + dd
        #print(d_)
        R = (sig_ * pow(msg,-d_,n_)) % n_
        for _ in range(j):
            R = pow(R,(n_+3)//4,n_)
            
        if (pow(R,2**j,n) * pow(msg,d_,n)) % n == sig or (pow(n_-R,2**j,n) * pow(msg,d_,n)) % n == sig:
            dd = d_
            print("[+]",dd)
            break
    else:
        print("[-] error")
        exit()

print("[+] part of d",dd)

```



```python
from Crypto.Util.number import *
from Crypto.Cipher import AES
from hashlib import md5
import random

def decrypt(message,key):
    key = bytes.fromhex(md5(str(key).encode()).hexdigest())
    enc = AES.new(key,mode=AES.MODE_ECB)
    c   = enc.decrypt(message)
    return c

def recover_p(p0, n,d0bits):
    PR.<x> = PolynomialRing(Zmod(n))
    nbits = n.nbits()
    p0bits = p0.nbits()
    f = 2^p0bits*x + p0
    f = f.monic()
    roots = f.small_roots(X=2^(nbits//2-p0bits+10), beta=0.4)  
    #print(roots)
    if roots:
        x0 = roots[0]
        p = gcd(2^d0bits*x0 + p0, n)
        return ZZ(p)

    
def find_p0(d0, e, n):
    X = var('X')
    for k in range(1, e+1):
        results = solve_mod([e*d0*X == k*n*X + k*X + X-k*X**2 - k*n], 2^d0.nbits())
        #print(results)
        for x in results:

            p0 = ZZ(x[0])
            #print(p0.nbits())
            p = recover_p(p0, n, d0.nbits())
            if p and p != 1:
                return p

from Crypto.Util.number import *
            

e = 17

n = 73468676168622364284797821322152865781682180355500517056745668297165363555301096288101727335605000017268011253011119083984169415642723459174940543244081395676001820974756949196348450763644469053660647326752189360358283884531064841995272730690738485358497415320211291684980496435204196241103362533593881904523


d0 = 62760647033001752398354920856119694316831379748195717723129573742917034443985946161152177


cipher = b'm\xcc\xed\xd9m?x}\x00\xdf\x85\x07jk\xefw\xbc\xb5i+\xcfpi\xf2]\x81#\xd0\xcc\x17<\x98\x15\x0ei\xea\xde\xe7\xadm\x8d\xff\xe5g\xe52"v'

p = int(find_p0(d0, e, n))
q = n//int(p)
d = inverse_mod(e, (p-1)*(q-1))
print(decrypt(cipher,d))

```

## 方法总结

- 核心技巧：对 RSA 模幂过程中的公钥模数做故障注入，通过正确签名和故障签名的代数关系逐位恢复私钥指数。
- 识别信号：签名服务允许“改变 N 的一个字节”并指定干扰位置 `j`，同时返回同一固定消息的签名时，应考虑 RSA public key perturbation fault attack。
- 复用要点：Left-to-Right 变体中要让故障模数 $\hat N$ 易于开方或判定二次剩余；低位恢复到足够多后，可结合 Coppersmith/已知低位分解 RSA。
