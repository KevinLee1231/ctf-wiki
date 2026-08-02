# make-groups

## 题目简述

题目有 $n=500000$ 个人和 $n$ 个相互独立的组，第 $i$ 个组必须选择 $a_i$ 个人，同一个人可以同时出现在多个组中。答案为所有组选择方案数之积，并对素数 $998244353$ 取模。原程序递归计算巨大阶乘后再做整数除法，不仅递归深度溢出，也完全无法承受这些整数的规模。

## 解题过程

单个大小为 $a_i$ 的组有 $\binom{n}{a_i}$ 种选法。各组独立，所以总数为

$$\prod_i\binom{n}{a_i}\pmod{998244353}.$$

预处理阶乘和逆阶乘后，每个组合数可在 $O(1)$ 时间求出。因为模数是素数且 $n<998244353$，可用费马小定理计算

$$n!^{-1}\equiv(n!)^{p-2}\pmod p,$$

再从高到低递推所有逆阶乘。

```python
with open("chall.txt", "r", encoding="utf-8") as f:
    n = int(f.readline())
    sizes = list(map(int, f.readline().split()))

mod = 998244353
factorial = [1] * (n + 1)
inverse_factorial = [1] * (n + 1)

for i in range(1, n + 1):
    factorial[i] = factorial[i - 1] * i % mod

inverse_factorial[n] = pow(factorial[n], mod - 2, mod)
for i in range(n, 0, -1):
    inverse_factorial[i - 1] = inverse_factorial[i] * i % mod

answer = 1
for size in sizes:
    combinations = (
        factorial[n]
        * inverse_factorial[size]
        * inverse_factorial[n - size]
    ) % mod
    answer = answer * combinations % mod

print(f"tjctf{{{answer}}}")
```

对仓库中的 500000 个组运行，得到：

```text
tjctf{148042038}
```

## 方法总结

- 核心技巧：把独立分组计数写成组合数乘积，并用阶乘/逆阶乘预处理降到 $O(n)$。
- 识别信号：大量重复查询同一个 $n$ 下的 $\binom{n}{r}$，且答案要求对大素数取模。
- 复用要点：模运算下不能先做普通除法；应乘模逆元。递归阶乘既会爆栈，也会产生没有必要的超大整数。
