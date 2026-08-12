# DownUnderCTF 2022 rsa interval oracle ii Writeup

## 题目简述

第二题仍公开 384 位 RSA 的 $(N,e,c)$，但只允许添加一个区间和使用 384 次 oracle。秘密只有 42 字节，即最多 336 位。选择固定区间 $(0,B)$，其中 $B=2^{376}$，即可把服务转化为一个阈值 oracle：它告诉我们查询密文的明文是否小于 $B$。

RSA 的乘法性质允许构造：

$$
c_f=c\cdot f^e\bmod N,
$$

其解密结果为 $f m\bmod N$。这正是 Manger attack 所需的 oracle 形式。

## 解题过程

定义 `outside(f)` 表示 $fm\bmod N\notin(0,B)$。攻击分三步：

1. 令 $f_1$ 不断乘 2，直到乘积首次越过 $B$。由于 $m<2^{336}$，可直接从 $f_1=2^{40}$ 开始，节省约 40 次查询。
2. 从 $f_1/2$ 的适当倍数开始增加 $f_2$，寻找一次模 $N$ 回绕后重新落入 $(0,B)$ 的位置。由此得到初始界：

   $$
   \left\lceil\frac{N}{f_2}\right\rceil
   \le m\le
   \left\lfloor\frac{N+B}{f_2}\right\rfloor.
   $$

3. 反复选择乘数 $f_3$，使当前候选区间经缩放后贴近某个 $iN$ 边界。根据 oracle 结果更新上界或下界，直到区间足够小。

核心查询函数如下：

```python
def outside(f):
    transformed = c * pow(f, e, N) % N
    return query(transformed) == -1
```

官方 solver 在剩余区间不超过 $2^{24}$ 时停止格外收缩，逐个验证候选是否满足 `pow(candidate, e, N) == c`，避免边界取整错误。最终提交正确秘密并得到：

```text
DUCTF{Manger_w0uld_b3_pr0ud_0f_y0u}
```

## 方法总结

固定单区间并不意味着泄漏无用；RSA 的乘法可塑性可以把同一个秘密缩放到阈值两侧，形成 Manger attack。查询预算紧张时，要利用明文实际位长小于模数这一先验来跳过确定无信息的前导倍增阶段。所有上下界更新都涉及严格不等式和向上/向下取整，最后应始终用公钥方程验证候选。
