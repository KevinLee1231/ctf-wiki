# 逐望

## 题目简述

题目使用基于椭圆曲线点的线性随机数生成器（EC-LCG），向选手输出一个 2025 位整数。根据 PDF 中保留的 exploit，可以确定该整数由 4 次 `_rand550()` 拼接而成，其中高位的最后一块被截断，而最低 1100 位包含两个完整的 550 位输出。每个完整输出都编码了一组被小幅扰动的椭圆曲线点坐标。

仓库没有保留本题源码，因此无法从附件验证原稿关于扰动具体来源的描述。可确认的是，官方 exploit 把两个坐标误差都界定为小于 $2^{36}$，并用二元 Coppersmith 恢复真实曲线点；以下分析以这一可执行约束为准。

## 解题过程

### 拆出两个完整的点编码

令服务输出为 $n$，最低两个 550 位分块为：

$$
n_0=n\bmod 2^{550},\qquad
n_1=\left\lfloor\frac{n}{2^{550}}\right\rfloor\bmod2^{550}
$$

每个分块转成 70 字节后分为两个 35 字节半块。对每个半块逐字节异或 `0x20`，丢弃开头 3 字节，再按小端序解释剩余 32 字节，就得到近似坐标 $(\hat{x},\hat{y})$。

### 用二元小根修正坐标

设真实坐标为：

$$
x=\hat{x}+\delta_x,\qquad
y=\hat{y}+\delta_y,
\qquad |\delta_x|,|\delta_y|<2^{36}
$$

曲线参数公开，真实点满足 $y^2=x^3+ax+b\pmod p$，于是构造二元多项式：

$$
f(\delta_x,\delta_y)=
(\hat{x}+\delta_x)^3+a(\hat{x}+\delta_x)+b-(\hat{y}+\delta_y)^2
\equiv0\pmod p
$$

脚本构造 $p^{m-i}f^i$ 与变量移位组成的格，按根界缩放列后执行 LLL，再把约化向量还原为多项式组，并通过零维理想求共同小根。对 $n_0,n_1$ 分别执行一次，即可得到两个真实点 $P_0,P_1$。

### 反推 EC-LCG 状态

相邻输出点之差为固定步长：

$$
Δ=P_1-P_0
$$

从 $P_0$ 开始反向迭代 `cur -= Δ`，检查每个点的 $x$ 坐标大端字节串。找到以 `0xGGGGame#` 开头的坐标后，取前 24 字节作为服务要求的 code 提交。

完整 exploit 如下。原 PDF 中的一次性远程地址已移除；运行时通过 pwntools 参数传入 `HOST` 和 `PORT`：

```sage
__import__('os').environ['TERM'] = 'xterm'
from pwn import *
from sage.all import *
import itertools
def small_roots(f, bounds, m=1, d=None):
    if not d:
        d = f.degree()
    R = f.base_ring()
    N = R.cardinality()
    f /= f.coefficients().pop(0)
    f = f.change_ring(ZZ)
    G = Sequence([], f.parent())
    for i in range(m + 1):
        base = N ** (m - i) * f ** i
        for shifts in itertools.product(range(d), repeat=f.nvariables()):
            g = base * prod(map(power, f.variables(), shifts))
            G.append(g)
    B, monomials = G.coefficient_matrix()
    monomials = vector(monomials)
    factors = [monomial(*bounds) for monomial in monomials]
    for i, factor in enumerate(factors):
        B.rescale_col(i, factor)
    B = B.dense_matrix().LLL()
    B = B.change_ring(QQ)
    for i, factor in enumerate(factors):
        B.rescale_col(i, 1 / factor)
    H = Sequence([], f.parent().change_ring(QQ))
    for h in filter(None, B * monomials):
        H.append(h)
        I = H.ideal()
        if I.dimension() == -1:
            H.pop()
        elif I.dimension() == 0:
            roots = []
            for root in I.variety(ring=ZZ):
                root = tuple(R(root[var]) for var in f.variables())
                roots.append(root)
            return roots
    return []
if not args.HOST or not args.PORT:
    raise SystemExit("usage: sage exp.sage HOST=<host> PORT=<port>")
io = remote(args.HOST, int(args.PORT))
# Curve parameters
p = 76884956397045344220809746629001649093037950200943055203735601445031516197751
a = 56698187605326110043627228396178346077120614539475214109386828188763884139993
b = 17577232497321838841075697789794520262950426058923084567046852300633325438902
E = EllipticCurve(Zmod(p),[a,b])
io.recvuntil(b'number:')
n = int(io.recvline().strip().decode())
# 2025 invokes getrandbits 4 times, 'former' three of which are not truncated
mask = 2**550 - 1
n0 = n & mask
n1 = (n >> 550) & mask
def extract_xy(num):
    bt = int(num).to_bytes(70, 'big')
    x = int.from_bytes(xor(bt[:35],b' ')[3:],'little')
    y = int.from_bytes(xor(bt[35:],b' ')[3:],'little')
    return x,y
def force(xh,yh):
    P = PolynomialRing(GF(p),names='dx,dy')
    dx,dy = P._first_ngens(2)
    x = xh + dx
    y = yh + dy
    f = x**3 + a*x + b - y**2
    _x,_y = small_roots(f,(2**36,2**36),2)[0]
    return xh+_x,yh+_y
x0,y0 = force(*extract_xy(n0))
x1,y1 = force(*extract_xy(n1))
P0 = E(x0,y0)
P1 = E(x1,y1)
dif = P1 - P0
cur = P0
while True:
    sc = int.to_bytes(int(cur.xy()[0]), 32, 'big')
    if sc.startswith(b'0xGGGGame#'):
        io.sendlineafter(b'code:',sc[:24])
        io.interactive()
        break
    cur -= dif
```

现有 PDF 没有记录服务返回的最终 flag，仓库也没有本题附件或 `secret.py`，因此不能从离线证据可靠补出比赛 flag；脚本最后进入交互模式，成功提交 code 后由服务输出。

## 方法总结

本题的主线是“截断的复合随机数 → 两个完整分块 → 带小误差的曲线坐标 → 二元 Coppersmith → 相邻 EC-LCG 点”。处理类似题目时，应先从总位数和分块宽度判断哪些调用结果未被截断，再把编码误差写进公开代数关系。多元小根并不是简单地“多爆几个未知量”，其可行性依赖模数、次数、根界和所构造格的维度；本题可复现的关键参数是二元曲线方程、两个 $2^{36}$ 根界以及 `m=2` 的格构造。
