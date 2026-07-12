# Sum

## 题目简述

本题是低密度子集和问题。题目给出 180 个数 $A=\{a_0,a_1,\ldots,a_{179}\}$ 和目标和 $s$，其中 $s$ 是 160 个元素的和，密度 $d=0.8$。直接从 180 个元素中枚举 160 个等价于枚举补集中的 20 个元素，规模为 $\binom{180}{20}\approx 2^{88}$，不可直接爆破。解法是把低密度子集和转化为格上的短向量问题，并用 Zero-Forced Lattices 降维后反复运行 BKZ，最终恢复 0/1 选择向量。

## 解题过程

子集和目标是找到 0/1 向量：

$$
\mathbf{m}=\{m_0,m_1,\ldots,m_{179}\},\quad m_i\in\{0,1\}
$$

满足：

$$
\sum_{i=0}^{179}m_i a_i=s
$$

原始解中有 160 个 `1`。为了降低汉明重量，PDF 中先反转 0 和 1，即改为求补集：

$$
s'=\sum_{i=0}^{179}a_i-s
$$

这样新的选择向量只有 20 个 `1` 和 160 个 `0`。

对低密度子集和，可以构造如下格基。令 $k$ 为目标向量汉明重量，$N$ 是平衡常数并满足 $N>\sqrt n$：

$$
B=
\begin{bmatrix}
1&0&\cdots&0&Na_0&N\\
0&1&\cdots&0&Na_1&N\\
\vdots&\vdots&\ddots&\vdots&\vdots&\vdots\\
0&0&\cdots&1&Na_{n-1}&N\\
0&0&\cdots&0&-Ns&-Nk
\end{bmatrix}
$$

如果某个 0/1 向量正好满足子集和与汉明重量约束，那么组合对应行后可以得到短向量：

$$
\mathbf{m}^*=\{m_0,m_1,\ldots,m_{n-1},0,0\}
$$

它的范数约为 $\sqrt{k}$。本题补集后 $k=20$，所以目标短向量非常短，BKZ/LLL 有机会把它规约出来。

实际难点是维度 $n=180$ 仍然偏高，直接反复 BKZ 成功率很低。PDF 采用 Zero-Forced Lattices：随机挑出 $r$ 个坐标，假设这些坐标在目标向量中为 `0`，并把对应列/元素从格中移除，从而降低维度。因为补集解中有 160 个 `0`，随机强制 1 个坐标为 0 的成功概率是：

$$
\frac{\binom{160}{1}}{\binom{180}{1}}=\frac{8}{9}
$$

若强制 $r$ 个坐标为 0，命中概率为：

$$
\frac{\binom{160}{r}}{\binom{180}{r}}
$$

$r$ 越大，降维越明显，但保留目标短向量的概率越低。PDF 中最终取 $r=40$，再对不同随机删列结果反复运行 BKZ。

关键 SageMath 逻辑如下。代码中最后一行使用正号 `[N*s, k*N]`，与上面的负号矩阵等价，因为格允许取负整数组合：

```python
def check(sol, A, s):
    return sum(x * a for x, a in zip(sol, A)) == s

def solve(A, n, k, s, r, ID=None, BS=22):
    N = ceil(sqrt(n))
    rand = random.Random(x=ID)
    indexes = set(range(n))

    while True:
        kick_out = set(sample(range(n), r))
        new_set = [A[i] for i in indexes - kick_out]

        lat = []
        for i, a in enumerate(new_set):
            lat.append([1 * (j == i) for j in range(n - r)] + [N * a] + [N])
        lat.append([0] * (n - r) + [N * s] + [k * N])

        shuffle(lat, random=rand.random)
        m = matrix(ZZ, lat)
        reduced = m.BKZ(block_size=BS)

        for row in reduced:
            if check(row, new_set, s) and row.norm() ^ 2 == k:
                return True
```

主函数中使用的参数为：

```python
k, n, d = 160, 180, 0.8
s, A = load(open("data", "r"))
r = 40

new_k = n - k
new_s = sum(A) - s
solve_n = partial(solve, A, n, new_k, new_s, r)
```

为了提高成功率，脚本用多进程并行反复打乱格基、运行 BKZ。PDF 中记录的实际情况是：总共运行了 30000 多次格规约，约 300+ CPU 小时后找到目标短向量。最终 flag 为：

```text
WMCTF{1508363197285327134921463070467158008637697619610562046}
```

## 方法总结

低密度子集和的核心信号是元素数量不大、目标为 0/1 选择向量、密度低且汉明重量可利用。若原向量汉明重量很大，可以先取补集降低重量。本题把 160 个 `1` 转成 20 个 `1`，再用格基把“和为 $s$、重量为 $k$”两个条件压到最后两列，使正确选择向量成为极短向量。维度过高时使用 Zero-Forced Lattices 随机强制部分 0 坐标降维，再通过多核反复 BKZ 搜索。
