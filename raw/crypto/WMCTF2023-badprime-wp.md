# badprime

## 题目简述

题目是 RSA 弱素数生成问题。源码中 `p` 不是普通随机素数，而是形如：

```python
p = int(pow(65537, getRandomRange(M >> 1, M), M)) + k * M
```

也就是说 `p mod M` 落在由 `65537^r mod M` 生成的子群中；同时服务端允许输入 `leak`，若 `leak` 是 `M` 的因子，就返回 `p % leak`。这些信息使得 `p` 的一部分同余类可被构造出来，再配合 Coppersmith/LLL 找到完整因子。

## 解题过程

这是 CVE-2017-15361 的一个变体。它通过将 65537 的阶作为较小 M 的因子提交上去实现该特性，从而使某些子群阶更小，分解成功率更高。

```python
from sage.doctest.util import Timer
t = Timer()

L = 0x7cda79f57f60a9b65478052f383ad7dadb714b4f4ac069997c7ff23d34d075fca08fdf20f95fbc5f0a981d65c3a3ee7ff74d769da52e948d6b0270dd736ef61fa99a54f80fb22091b055885dc22b9f17562778dfb2aeac87f51de339f71731d207c0af3244d35129feba028a48402247f4ba1d2b6d0755baff6
g = Mod(65537,L)

pmin = 3*2**1022
pmax = 4*2**1022

p = 119949297823304007163602750328870391606548779718070065324766633638259841939589549994095387946619248438497339849221415471645532921809358722871360231737322831463096854247759249766367829886583523947636952483726472899501770613135658534476120556587962031682504046820345732516331262954429216339141280964449499359983
n = 19807826992583521250431605870196413245365136314869741490002104280652366308632165313760212277782501214004565242950974066601397923682737778720436468720382474478235064191521362420031564770913641262988586682000357716497592630218790724907173726965450371494113979140305229061655699467992501647949265532733053410019415270972975479654141623451913791822360781514630565929921143757567864778890510836686621302319405359981077678783318003917128556117114849777515891030325901596390420589871218059230490823503152102165169768689027574992715596780402524541734444046121490095248121766937156745199111285909451067406877297015574617767327
print ('public key',n)

smooth = 2^7*3^3*5^2*7*11*13*17*19*23
print ('smooth',smooth)
def smoothorder(l):
  return smooth % Mod(g,l).multiplicative_order() == 0

v = prod(l for l,e in factor(L) if smoothorder(l))
print (v)
u = p % v
print ('p residue class',(p-u)/v)

t.start()

H = 10 + 2**1021 // v
u += floor((7*2**1021) // v) * v

w = lift(1/Mod(v,n))

R.<x> = QQ[]
f = (w*u+H*x)/n
g = H*x

k = 3
m = 7
print ('multiplicity',k)
print ('lattice rank',m)

basis = [f^j for j in range(0,k)] + [f^k*g^j for j in range(m-k)]
basis = [b*n^k for b in basis]
basis = [b.change_ring(ZZ) for b in basis]

M = matrix(m)
for i in range(m):
  M[i] = basis[i].coefficients(sparse=False) + [0]*(m-1-i)
print ('time for creating matrix',t.stop().cputime)

t.start()
M = M.LLL()
print ('time for basis reduction',t.stop().cputime)

Q = sum(z*(x/H)^i for i,z in enumerate(M[0]))

for r,multiplicity in Q.roots():
  print ('root is',r)
  if u+v*r > 0:
    g = gcd(n,u+v*r)
    if g > 1: print ('successful factorization',[g,n/g])
```
成功分解 n 的可能性更大

脚本的核心是从 `M` 的因子中筛选出满足 `65537` 在该因子模数下阶较光滑的部分，构造较大的模数因子 `v`，并利用交互泄露得到 `u = p mod v`。已知 `p = u + v*x` 且 `p` 位于 1024 bit 范围内，就可以把未知量 `x` 变成小根问题。格基约化得到候选根后，用 `gcd(n, u + v*r)` 验证是否恢复出 `p`。

## 方法总结

- 核心技巧：利用 RSA 素数生成中的 ROCA/CVE-2017-15361 类结构，使 `p` 落入可预测同余类，再用 Coppersmith 小根恢复因子。
- 识别信号：素数生成出现 `pow(e, random, M) + k*M`、服务端泄露 `p mod leak`、且 `leak` 必须整除固定 `M` 时，应考虑构造 `p mod v`。
- 复用要点：同余信息越大，小根搜索越容易；选择 `M` 的哪些因子时要关注 `65537` 的阶是否光滑。
