# inverse pow

## 题目简述

服务每轮给出随机前缀 `m`，要求找到 `n`，使 `2^n` 的十进制表示以 `m` 开头。单轮程序用 Go 的 `big.Int.Exp` 直接验证，远端包装脚本连续跑 8 轮，全部通过后输出 flag。

## 解题过程

附件中的核心逻辑很短：

```go
m, _ := rand.Int(rand.Reader, big.NewInt(1e8-1))
m = m.Add(m, big.NewInt(1))
fmt.Printf("m = %d\n", m)

go func() {
    time.Sleep(60 * time.Second)
    fmt.Println("TimeOut")
    os.Exit(1)
}()

n, _ := strconv.Atoi(strings.TrimSpace(input))
power := new(big.Int).Exp(big.NewInt(2), big.NewInt(int64(n)), nil)
if strings.HasPrefix(power.String(), m.String()) {
    fmt.Println("Verified")
}
```

远端包装脚本只是重复 8 轮：

```bash
ROUNDS=8
for ((i=1; i<=$ROUNDS; i++)); do
    ./inverse_pow || exit $?
done
cat flag
```

因此关键不是大整数爆破，而是在 60 秒验证时间内找出足够小的 `n`。如果 `m` 有 `d` 位，`2^n` 以 `m` 开头等价于存在整数 `k`：

$$
m\cdot 10^k \le 2^n < (m+1)\cdot 10^k
$$

两边取 $\log_{10}$ 后，条件变成：

$$
\{n\log_{10}2\}\in
\left[\log_{10}(m)-(d-1),\ \log_{10}(m+1)-(d-1)\right)
$$

也就是在模 1 意义下让 $n\log_{10}2$ 的小数部分命中一个很窄的区间。这个问题可以用 LLL 找短关系，也可以用 baby-step giant-step 做高精度查表。为了控制远端验证耗时，实际需要优先找较小的 `n`。

查表实现把小数部分映射到 $2^{128}$ 固定点环：

```python
SCALE = 1 << 128
LOG10_2_I = int((Decimal(2).ln() / Decimal(10).ln()) * SCALE)

def prefix_interval(m):
    d = len(str(m))
    lo = log10(m) - (d - 1)
    hi = log10(m + 1) - (d - 1)
    return int(lo * SCALE), int(hi * SCALE)
```

设最大搜索范围为 `max_n`，取 `B = ceil(sqrt(max_n))`。预计算 baby step：

```python
babies = [((j * LOG10_2_I) % SCALE, j) for j in range(B)]
babies.sort()
```

再枚举 giant step，把目标区间平移回来后在 `babies` 中二分查找：

```python
def solve_prefix(m, max_n):
    B = isqrt(max_n) + 1
    babies = [((j * LOG10_2_I) % SCALE, j) for j in range(B)]
    babies.sort()
    values = [x for x, _ in babies]
    step = (B * LOG10_2_I) % SCALE
    target_lo, target_hi = prefix_interval(m)

    giant = 0
    for i in range((max_n + B - 1) // B + 1):
        lo = (target_lo - giant) % SCALE
        hi = (target_hi - giant) % SCALE
        ranges = [(lo, hi)] if lo < hi else [(lo, SCALE), (0, hi)]
        for begin, end in ranges:
            idx = bisect_left(values, begin)
            if idx < len(values) and values[idx] < end:
                n = i * B + babies[idx][1]
                if n <= max_n and target_lo <= (n * LOG10_2_I) % SCALE < target_hi:
                    return n
        giant = (giant + step) % SCALE
    return None
```

本地附件验证过一组样例：

```text
m = 24030749
n = 21407353
Verified
```

远端的瓶颈在服务端计算 `2^n`。本地测试中 `n=80,000,000` 约 18.58 秒，`n=100,000,000` 约 26.64 秒，`n=120,000,000` 约 44.32 秒，`n=140,000,000` 约 55.02 秒。为了留出网络和调度余量，远端脚本把 `--max-n` 设为 `120000000`；如果某轮找不到足够小的 `n`，就断开当前连接，等待限速后重试。

一次成功的 8 轮结果为：

```text
ROUND 1: m=35353872, n=55224163
ROUND 2: m=79171855, n=48606611
ROUND 3: m=17088504, n=19325550
ROUND 4: m=9701749,  n=14884613
ROUND 5: m=49389957, n=92829705
ROUND 6: m=10014606, n=7030904
ROUND 7: m=38134187, n=29809719
ROUND 8: m=7001519,  n=23853154
```

运行时把队伍名和 token 作为参数或环境变量传入即可：

```bash
INVERSE_POW_TEAM="<team>" \
INVERSE_POW_TOKEN="<token>" \
python3 solve_inverse_pow.py --attempts 6 --rounds 8 --max-n 120000000 --verify-timeout 220
```

最终得到：

```text
ACTF{h0w2F1nDnUm@BV1mj7Wz6EXp|DbiFyzKH4Ra}
```

## 方法总结

- 核心技巧：把十进制前缀匹配转成 $\{n\log_{10}2\}$ 的区间命中问题。
- 识别信号：构造幂的十进制前缀时，不要直接爆大整数，先取对数转换为模 1 区间问题。
- 复用要点：多轮服务要限制 `n` 的大小，否则本地求解很快，远端验证 `2^n` 反而会超时。
