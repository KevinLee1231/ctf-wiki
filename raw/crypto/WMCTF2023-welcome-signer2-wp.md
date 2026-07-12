# welcome_signer2

## 题目简述

题目延续 welcome_signer1：服务生成 RSA 密钥，`G` 返回 `n` 和 AES 加密 flag，`F` 允许把 `N` 的一字节翻转为故障模数 $\hat N$，`S` 返回固定消息在指定干扰位置下的故障签名。不同点在于模幂实现换成 Right-to-Left 形式，因此可从高位方向逐步恢复私钥指数。

外部论文《Perturbating RSA Public Keys: an Improved Attack》的关键思路是：在 Right-to-Left 模幂中，故障模数会影响后续 `B = B^2 mod N` 链；攻击者用正确签名 $S$、故障签名 $\hat S$、原模数 $N$、故障模数 $\hat N$ 和可控位置 $j$，枚举当前未知私钥位并验证计算出的故障签名是否匹配，从而从高位向低位恢复 `d`。

## 解题过程

参考：《Perturbating RSA Public Keys: an Improved Attack》

简要描述：使用从右到左算法计算签名。

```python
def Right_to_Left(self,j):
    A = 1
    B = self.m
    d = self.d.bits()
    n = len(d)
    N = self.N
    for i in range(n):
        if d[i] == 1:
            A = A * B % N
        B = B**2 % N
    return A
```
错误注入模型

```python
def fault_model(self,j):
    A = 1
    B = self.m
    d = self.d.bits()
    n = len(d)
    N = self.N
    for i in range(n):
        if d[i] == 1:
            A = A * B % N
            #  a fault occurs j steps before the end of the exponentiation
        if i >= n-1-j:
            N = self.N_
        B = B**2 % N
    return A
```
正确的签名为
$$
S \equiv m^{\sum_{i=0}^{n-1}2^i\cdot d_i} \pmod N
$$
错误注入后，错误的 B 的表达式为
$$
\hat{B} \equiv (m^{2^{n-j-i}} \mod N)^2 \mod \hat N
$$
因此错误的签名为
$$
\hat S \equiv ((A\cdot \hat B)\dots)\hat B^{2^{j-1}} \\
\equiv A\cdot \hat B^{\sum_{i=(n-j)}^{n-1}2^{[i-(n-j)]}} \mod \hat N \\
\equiv [(m^{\sum_{i=0}^{(n-j-i)}2^i\cdot d_i} \mod N)\cdot (m^{2^{n-j-1}} \mod N)^{\sum_{i=(n-j)}^{n-1}2^{[i-(n-j)+1]}\cdot d_i}] \mod \hat N 
$$
攻击条件：已知 $N,\hat N,S$，已知注入位置 $j$（可控），由于高位爆破私钥 $d'$，计算签名，
$$
S' \equiv [((S\cdot m^{-d'}) \mod N)\cdot (m^{2^{(n-j-1)} } \mod N)^{2^{[1-(n-j)]\cdot }d'}] \mod \hat N
$$
验证是否成立
$$
S' \equiv \hat S \mod \hat N
$$
以判断爆破出的 $d'$ 是否正确。

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


def decrypt(message,key):
    key = bytes.fromhex(md5(str(key).encode()).hexdigest())
    enc = AES.new(key,mode=AES.MODE_ECB)
    c   = enc.decrypt(message)
    return c

msg = bytes_to_long(b"Welcome_come_to_WMCTF")

# sh = process(["python","task.py"])
sh = remote("0.0.0.0",9999)
sh.recvuntil("[Q]uit\n")
sh.sendline("G")
sh.recvuntil("| n = ")
n = int(sh.recvuntil("\n")[:-1])
sh.recvuntil("ciphertext = ")
cipher = bytes.fromhex(sh.recvuntil("\n").decode()[:-1])



index = random.randint(0,1024)
temp = random.randint(0,256)
n_ = n ^ (temp<<index)

sig = get_sig(0)



sh.recvuntil("[Q]uit\n")
sh.sendline("F")
sh.recvuntil("index:")
sh.sendline(",".join([str(temp),str(index)]))

# poc ps: associate with coppersmith will be more efficient

d = 0
j = 1
sig_ = get_sig(j)

length = n.bit_length()
for offset in range(8):
    i=1
    d_ = (i << (length-j)) + d
    check = int(sig * pow(msg,-d_,n)%n) * pow(pow(msg,2**(length-j-1),n),(d_>>(length-j-1)),n_) % n_
    if check == sig_:
        d = d_
        print("[+]",d)
        break
    else:
        length -= 1


for j in range(2,length):
    sig_ = get_sig(j)
    for i in range(2):
        d_ = (i << (length-j)) + d
        check = int(sig * pow(msg,-d_,n)%n) * pow(pow(msg,2**(length-j-1),n),(d_>>(length-j-1)),n_) % n_
        if check == sig_:
            d = d_
            print("[+]",bin(d))
            break
    else:
        print("[-] error",j)
        exit()
d = d + 1

print(decrypt(cipher,d))


```

## 方法总结

- 核心技巧：针对 Right-to-Left RSA 模幂的 public modulus fault attack，根据故障发生位置从高位恢复私钥指数。
- 识别信号：模幂实现维护 `A` 和 `B` 两个变量，且故障在接近指数末尾时切换到 $\hat N$，应分析 `B` 链对后续乘法的影响。
- 复用要点：攻击验证条件来自“候选私钥位推导出的故障签名是否等于 oracle 返回值”；恢复完整 `d` 后直接用题目给定的 `md5(str(d))` 派生 AES key 解密即可。
