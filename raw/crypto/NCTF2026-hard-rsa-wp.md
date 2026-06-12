# Hard_RSA

## 题目简述

RSA 题。官方 WP 提到预期参考论文 [A Generalized Wiener-type Attack Against an RSA-like Cryptosystem](https://eprint.iacr.org/2025/1396)，本地论文文件为 `D:/文档/新建文件夹/2025-1396.pdf`。论文研究的是一类 RSA-like 密码体制：模数 `N = pq`，群阶不再是传统 `phi(N) = (p - 1)(q - 1)`，而是 `phi_n(N) = (p^n - 1)(q^n - 1)`；其中 `n = 6` 时可做 generalized Wiener-type attack。

官方原始 WP 只写了“论文题”和“普通 Wiener 扩展思路能做”，没有给出 exploit。这里补充论文主线，方便后续只读 WP 也能理解预期攻击方向。

## 解题过程

论文模型来自 generalized Elkamchouchi/RSA-like scheme。密钥方程是：

```text
ed - k * phi_6(N) = 1
phi_6(N) = (p^6 - 1)(q^6 - 1)
N = pq, q < p < 2q
```

论文把攻击写成更一般的形式：

```text
a * e - b * phi_6(N) = c
```

核心思想分两步。

### 连分数恢复 a 和 b

论文证明，在 `|c| < b * N * sqrt(N)` 且满足界：

```text
a * b < (N^6 - 162N^3 + 1) / (2N*sqrt(N) + 1100N^3)
```

时，`b / a` 会出现在下面这个数的连分数收敛分数中：

```text
e / (N^6 - 162N^3 + 1)
```

所以攻击时先枚举该连分数的 convergents，把每个分数当作候选 `b / a`。

对真正的小私钥场景 `ed - k * phi_6(N) = 1`，可以对应到上面的式子里：`a = d`、`b = k`、`c = 1`。如果 `d` 足够小，`k*d` 满足论文给出的乘积界，就可以从连分数中恢复候选。

### 用 p 的近似值分解 N

拿到候选 `a, b` 后，论文通过 `phi_6(N)` 的展开式构造 `p` 的近似值。设：

```text
S = p + q
D = p - q
```

对 `phi_6(N)` 可写成关于 `S` 或 `D` 的形式：

```text
phi_6(N) = N^6 + 2N^3 - 9N^2S^2 + 6NS^4 - S^6 + 1
phi_6(N) = N^6 - 2N^3 - 9N^2D^2 - 6ND^4 - D^6 + 1
```

论文用 Cardano 公式推导出 `S` 与 `D` 的近似，再得到：

```text
p_hat = cbrt( 1/2 * (
    sqrt((N^3 - 1)^2 - a*e/b)
  + sqrt((N^3 + 1)^2 - a*e/b)
))
```

当候选 `a, b` 正确时，论文证明：

```text
|p - p_hat| < N^(1/4)
```

这就把问题转成“已知 `p` 的一个高精度近似值分解 `N`”。根据 Coppersmith/high-bits-known factoring 的结论，如果 `p` 的误差小于 `N^(1/4)`，就可以在多项式时间内恢复 `p`，进而得到 `q = N // p`。

论文实验中，作者对样例 `N` 和 `e` 做连分数展开，在第 87 个 convergent 得到：

```text
a = 11150372599265311570767859136324180752990213
b = 4006431015451326125727099215660449213022640
p_hat = 132287523294776463864922187740
p     = 132287523294776463864922187741
q     = 26379325137285943549540377391
```

这个例子说明 `p_hat` 与真实 `p` 只差 1，后续用 Coppersmith 或直接小范围修正都能分解模数。

### 本题处理方式

官方 WP 提到本题被非预期做掉，常规 Wiener 扩展思路已经足够。实际写 exploit 时可以按这个顺序尝试：

1. 先判断题目是否真的是 `phi_6(N) = (p^6 - 1)(q^6 - 1)` 这类 RSA-like 体制。如果给出了 `n = 6`、扩域/多项式环运算、或密钥方程形如 `ed ≡ 1 mod phi_6(N)`，就按论文路线做。
2. 枚举 `e / (N^6 - 162N^3 + 1)` 的 convergents，取候选 `b/a`。
3. 用候选 `a,b` 算 `p_hat`，再用 Coppersmith 或在 `p_hat` 附近搜索恢复 `p`。
4. 如果题目参数其实只是传统 RSA 的小 `d`，则不需要完整论文路线，直接试 Wiener、Wiener 扩展或 continued fraction 变体即可。

## 方法总结

- 核心技巧：generalized Wiener-type attack。先用连分数恢复密钥方程中的小参数，再由论文公式构造 `p` 的高精度近似，最后用 Coppersmith/high-bits-known factoring 分解 `N`。
- 识别信号：RSA-like 题出现 `phi_n(N) = (p^n - 1)(q^n - 1)`、`n = 6`、`ed - k*phi_6(N) = 1` 或论文 `2025/1396` 时，应想到该路线。
- 复用要点：外部论文不能只留链接；WP 中至少要保留攻击适用条件、连分数枚举对象、`p_hat` 公式和分解条件。若题目实际是更简单的小私钥 RSA，则先用普通 Wiener 扩展验证。
