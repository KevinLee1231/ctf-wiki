# Noise

## 题目简述

题目给出未知 1024-bit 整数 $n$，黑盒查询不超过 50 次：输入非负整数 $x$，返回 $f(x)=(x+r)\bmod n$，其中每次查询的噪声 $r$ 是重新生成的 1000-bit 随机数。目标是恢复 $n$。解法不是普通二分，而是维护 $n$ 的区间 $[L,R]$，构造查询点 $I$ 和整数 $t$，保证所有可能的 $n,r$ 都落在同一个商 $t$ 上，从而把一次查询转化为对新区间的强收缩。

题目资料地址：https://gist.github.com/Chrstm/f225a5e67f12d20caba117224d1b4241

服务端脚本 `noise.py` 先生成 `secret = getrandbits(1024)`，最多交互 50 轮；当 `op == 'god'` 时返回 `(num + getrandbits(1000)) % secret`，当 `op == 'bless'` 且 `num == secret` 时输出 flag。`noise_exp.py` 也明确使用 `NUM_BIT=1024`、`NOISE_BIT=1000`、`scope=1<<1000`，先做初始二分，再反复用固定商区间收缩 `[L,R]`，最后用 `bless` 验证候选。

## 解题过程

$n$ 是一个未知整数 (1024 bits)。你可以访问一个黑盒函数不超过 50 次：给定一个非负整数 $x$，返回 $f(x) = (x + r) \bmod n$，$r$ 是一个每次都重新生成的随机数 (1000 bits)。求 $n$。

一种比较自然的想法是二分查找，但这样每次只能获取到 1 bit 信息且当区间大小接近噪声大小时就无法再缩小范围了。不过我们可以以这种方式切入，假设已知某个范围并想办法缩小它。

假设我们已有 $[L, R]$ 满足 $n \in [L, R]$。如果我们可以找到一对 $(I, t)$ 对于所有可能的 $n$ 和 $r$ 都满足：

$$f(I) = (I + r) \bmod n = I + r - tn$$

即

$$tn = I - f(I) + r$$

那么显然 $I – f(I) + r$ 是 $t$ 的倍数，它的范围只和 $r$ 有关。所以可得 $n$ 的新区间：

$$n = \frac{I - f(I) + r}{t} \in \left[ \frac{I - f(I) + r_{\min}}{t},\ \frac{I - f(I) + r_{\max}}{t} \right]$$

$r$ 的范围是 $[0, 2^{1000}]$。令 $s = r_{\max} = 2^{1000}$，则上述区间长度大约是 $\frac{s}{t}$。

现在的问题就转化为如何选取一对合适的 $(I, t)$ 并且 $t$ 尽可能大，这样得到的新区间就尽可能小。回到一开始的假设上，

$$\forall r \in [0, s],\ n \in [L, R],\quad tn \leq I + r < (t+1)n$$

即

$$(tn - r)_{\max} \leq I < ((t + 1)n - r)_{\min}$$

则有

$$(tn - r)_{\max} = tR \quad < \quad ((t + 1)n - r)_{\min} = (t + 1)L - s$$

$$\Leftrightarrow \quad tR \leq (t + 1)L - s - 1$$

$$\Leftrightarrow \quad t \leq \frac{L - s - 1}{R - L}$$

只要 $t$ 满足这个条件，我们就可以找到对应的 $I$。除此之外，因为 $t$ 参与除法，所以还有一个下界 $t \geq 1$，由此我们可以得到对 $L, R$ 的限制条件:

$$1 \leq t \leq \frac{L - s - 1}{R - L} \quad \Rightarrow \quad 2L \geq R + s + 1$$

有两种方法来设置合法的初始 $[L, R]$：

1. 直接假设 $n$ 落在 $[2^{1023}, 2^{1024}]$（这样的概率大约是 50%，所以如果假设不成立就多试几次）
2. 先做少量二分查找，把 $n$ 的候选范围压到可用的初始区间；公开 [exp](https://gist.github.com/Chrstm/f225a5e67f12d20caba117224d1b4241#file-noise_exp-py) 也是按 `NUM_BIT=1024`、`NOISE_BIT=1000` 设定 `scope=2^1000`，再用固定商区间反复收缩 `[L,R]`。

综上所述，首先选取一对 $L, R$ 作为 $n$ 的初始范围。然后令 $t$ 为最大值 $\frac{L - s - 1}{R - L}$ 并在区间 $[tR,\ (t + 1)L - s - 1]$ 中选择一个 $I$，由此可得新区间

$$n \in \left[ \left\lceil \frac{I - f(I)}{t} \right\rceil,\ \left\lfloor \frac{I - f(I) + s}{t} \right\rfloor \right]$$

更新 $[L, R]$ 的值重复上述过程。$n$ 的范围跨度从 $\Delta \sim R-L$ 缩小到 $\Delta’ \sim \frac{s}{t}$，有

$$\Delta’ \sim \frac{s}{t} \sim \frac{s(R-L)}{L-s-1} \sim \frac{s\Delta}{L-s} \sim \frac{s\Delta}{n}$$

所以迭代次数大约是 $\log_s n$，一般在 45 次左右（同样如果一次不行，就多试几次）

FYI：这是一般性的做法。如果直接做一些假设（譬如假设 $n$ 落在某个范围，令 $I = tR$ 等等），可以省略掉一些繁琐的步骤得到一些更简单但需要多试几次的做法。

## 方法总结

- 核心技巧：为带随机噪声的取模 oracle 构造固定商区间，让 $f(I)$ 反推出 $n$ 的窄区间，而不是逐 bit 判断。
- 识别信号：返回值形如 `(x + small_random) mod unknown_modulus`，噪声范围已知且查询次数有限时，应考虑区间收缩和固定商构造。
- 复用要点：合法查询需要满足 $tR \le (t+1)L-s-1$；初始区间可从位长假设或少量二分得到，若位长假设失败就重新尝试。

