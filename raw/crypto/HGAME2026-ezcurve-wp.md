# ezCurve

## 题目简述

题面提示 flag 以 `hgame` 开头，附件是一个 Sage 椭圆曲线交互服务。每次连接服务端随机生成 1024 bit 素数域上的曲线、随机点 `R` 和隐藏点 `P`，并把 `p,a,b,R` 发给客户端。菜单 `1` 接收整数 `t`，返回 `P + tR` 的横坐标再减去一个 163 bit 随机素数；菜单 `2` 要求提交 `P[0]`。

```python
p = getPrime(1024)
a = getPrime(200)
b = getPrime(200)
E = EllipticCurve(GF(p), [a, b])
R = E.random_element()
P = E.random_element()

if choice == 1:
    t = int(istream.readline().strip())
    O = P + t * R
    print(int(O[0] - getPrime(163)), file=ostream)
elif choice == 2:
    ans = int(istream.readline().strip())
    if ans == P[0]:
        print(flag, file=ostream)
```

题目核心是这个带噪声的椭圆曲线横坐标查询 oracle。

## 解题过程

发送 `t` 和 `-t`，可以把响应组织成关于 `P`、`tR` 和 163 bit 噪声的关系。由于椭圆曲线上相反点横坐标相同、纵坐标互为相反数，成对查询能消掉不需要直接恢复的纵坐标部分，再把剩余小误差放进格中求解。

```sage
from Pwn4Sage.pwn import *
from Crypto.Util.number import *
#part1 get number
HOST = "challenge-host"  # 运行时替换为题目服务地址
PORT = 0
sh = remote(HOST, PORT)
p = int(sh.recvline().strip().decode())
a = int(sh.recvline().strip().decode())
b = int(sh.recvline().strip().decode())
R=sh.recvline().strip().decode()[1:-1]
x, y, z = map(int, R.split(" : "))
R = (x,y)
E = EllipticCurve(GF(p), [a, b])
R = E(R)
#part2 send t and -t to construct poly
h = []
xQ = [0]
#first get error_P
sh.recvuntil(b"> ")
sh.sendline(b"1")
sh.recvuntil(b"t> t = ")
sh.sendline(b"0")
h.append(int(sh.recvline().strip().decode()))
n = 15
for t in range(1,n):
    sh.recvuntil(b"> ")
    sh.sendline(b"1")
    sh.recvuntil(b"t> t = ")
    sh.sendline(str(t).encode())
    h1 = int(sh.recvline().strip().decode())
    sh.recvuntil(b"> ")
    sh.sendline(b"1")
    sh.recvuntil(b"t> t = ")
    sh.sendline(str(-t).encode())
    h2 = int(sh.recvline().strip().decode())
    h.append(h1+h2)
    xQ.append(int((t*R)[0]))
A = [h[i] - 2*xQ[i] for i in range(1,n)]
A0 = [2*(h[0]-xQ[i]) for i in range(1,n)]
B = [2*(h[i]*(h[0]-xQ[i])-2*h[0]*xQ[i]-a-xQ[i]^2) for i in range(1,n)]
B0 = [(h[0]-xQ[i])^2 for i in range(1,n)]
C = [
    h[i]*(h[0]-xQ[i])^2 - 2*((h[0]^2+a)*xQ[i] + (a+xQ[i]^2)*h[0] + 2*b)
    for i in range(1, n)
]
#part3 use poly to get Lattice and LLL
delta = 2^163
n = n-1
L = Matrix(ZZ,3*n+3,3*n+3)
L[0,0] = delta^3
for i in range(n+1):
    L[i+1,i+1] = delta^2
for i in range(n+1):
    L[i+n+2,i+n+2] = delta
for i in range(n):
    L[i+2*n+3,i+2*n+3] = p
for i in range(n):
    L[0,i+2*n+3] = C[i]
    L[1,i+2*n+3] = B[i]
    L[2+i,i+2*n+3] = B0[i]
for i in range(n):
    L[n+2,i+2*n+3] = A[i]
    L[n+2+1+i,i+2*n+3] = A0[i]
res = L.LLL()[0]
#part4 get flag
e0 = int(GCD(res[1],res[n+2]) // delta)
print(isPrime(e0))
xP = h[0] + e0
sh.recvuntil(b"> ")
sh.sendline(b"2")
sh.sendline(str(xP).encode())
print(sh.recvline())
print(sh.recvline())
```

## 方法总结

ECC oracle 题要优先利用点加法的对称性。若 `t` 和 `-t` 的响应能抵消纵坐标或噪声，就能把曲线问题降成多项式关系；小误差或隐藏量再用 LLL/GCD 等格方法恢复。
