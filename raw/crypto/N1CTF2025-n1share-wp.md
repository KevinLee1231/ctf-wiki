# N1CTF 2025 n1share

## 题目简述

服务端把 256 字节随机密钥分成两段，每段作为一个次数不超过 127 的多项式系数，在有限域 $\mathbb F_{521}$ 上计算 $x=1,\ldots,520$ 处的取值。两路 share 在同一组 300 个横坐标上同时被随机噪声破坏。最后以 `MD5(key)` 为 AES-128 密钥，用 ECB 模式加密 flag。

这相当于长度 $n=520$、维数 $k=128$ 的两路交织 Reed-Solomon 码，且两路错误位置完全同步。单独解码每一路最多只能处理 Berlekamp-Welch 的 196 个错误，普通 Guruswami-Sudan 界约为 $520-\sqrt{520\times128}\approx262$，都不足以纠正 300 个错误。

## 解题过程

把每个样本视为三维点

$$
P_i=(\alpha_i,y_{1,i},y_{2,i}),
$$

正确点位于参数曲线

$$
x\longmapsto \bigl(x,f_1(x),f_2(x)\bigr)
$$

上，而 300 个错误点在后两个坐标上偏离该曲线。这样就能利用“两路错误位置相同”这一额外信息，而不是分别解两个 RS 实例。

官方解法实现了 Coppersmith 与 Sudan 的三维噪声曲线重构方法。其核心是对变量 $(x,y_1,y_2)$ 建立带权多项式空间：$x$ 的权重为 1，$y_1,y_2$ 的权重均为 128。代码选择加权次数上界 $\ell=1090$、重数 $r=5$，枚举满足

$$
a+128b+128c\leq\ell
$$

的单项式 $x^a y_1^b y_2^c$。对每个接收点再枚举总阶小于 $r$ 的三元 Hasse 导数条件；相应多重指标数量为

$$
\#\{(i,j,k):i+j+k<5\}=\binom{7}{3}=35.
$$

构造矩阵时，每个样本占 35 列，每一行对应一个允许的带权单项式。官方代码通过平移多项式

```python
(x + alpha_i)**e0 * (y1 + value1_i)**e1 * (y2 + value2_i)**e2
```

提取各单项式系数，从而编码点上的重数约束。参数满足正确点数 $t=520-300=220>\ell/r=218$：若候选关系在这些正确点以重数 $r$ 消失，那么代入 $(x,f_1(x),f_2(x))$ 后所得一元多项式次数不超过 $\ell$，却拥有超过次数上界的零点重数，因此必须恒等为零。这正是能够从大量同步错误中恢复曲线的原因。

官方 Sage 脚本的矩阵与错误位置提取流程如下：

```python
p, n, r, ell = 521, 520, 5, 1090
Fp = GF(p)
R = PolynomialRing(Fp, ['x', 'y1', 'y2'])
x, y1, y2 = R.gens()

derivatives = [
    e for e in product(range(r), repeat=3)
    if sum(e) < r
]
monomials = [
    (a, b, c)
    for a in range(ell + 1)
    for b in range(ell // 128 + 1)
    for c in range(ell // 128 + 1)
    if a + 128*b + 128*c <= ell
]

# A 的每个点占 len(derivatives)=35 列；实际脚本分块并行构造。
kernel_vector = sum(A.right_kernel().matrix())

bad = []
for i in range(n):
    block = kernel_vector[i*35:(i+1)*35]
    if all(v == 0 for v in block):
        bad.append(i)
```

这里的右核向量是多重插值约束间的依赖关系；按点分块后，全零的 35 元块给出错误位置。去掉这些位置，剩下 220 个正确点已经远多于恢复次数不超过 127 的多项式所需的 128 个点。分别拉格朗日插值得到 $f_1,f_2$：

```python
PR = PolynomialRing(GF(521), 'x')
good1 = [(i + 1, shares[2*i]) for i in range(520) if i not in bad]
good2 = [(i + 1, shares[2*i + 1]) for i in range(520) if i not in bad]

f1 = PR.lagrange_polynomial(good1)
f2 = PR.lagrange_polynomial(good2)
part1 = bytes(int(f1[i]) for i in range(128))
part2 = bytes(int(f2[i]) for i in range(128))
key = part1 + part2
```

最后计算 `MD5(key)` 并解密：

```python
import hashlib
from Crypto.Cipher import AES

aes_key = hashlib.md5(key).digest()
flag = AES.new(aes_key, AES.MODE_ECB).decrypt(ciphertext)
```

算法依据是论文 [Reconstructing Curves in Three (and Higher) Dimensional Spaces from Noisy Data](https://people.csail.mit.edu/madhu/papers/2003/copper-conf.pdf)。论文的关键作用已经在上文展开：把同步的多路 RS 样本视为高维曲线点，用带权次数和高重数插值制造代数关系，再从约束依赖中区分错误点；复现本题不需要再从外链猜测参数。

## 方法总结

本题故意把错误数放在单路 RS 列表译码能力之外，决定性线索是两路 share 的 300 个错误位置完全一致。把两条取值序列联合成三维参数曲线后，额外坐标提供了足够约束。实现上最耗时的是大矩阵构造，可像官方脚本一样按列分块并行；但正文所需的逻辑只有四步：建立带权多项式空间、加入重数约束、用核向量定位坏点、在干净点上插值并恢复 AES 密钥。
