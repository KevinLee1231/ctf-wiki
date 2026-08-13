# GreyCTF 2023 OTP

## 题目简述

服务生成 6 个 16 位 OTP，并允许用户提交 6 个大整数 token。除了一组群元素，它还直接打印内积 $\sum k_i\theta_i$，其中 $k_i$ 是用户 token，$\theta_i$ 是 OTP。选择合适的 token 后，这个“调试值”就是 6 个 OTP 数字的进位制编码。

## 解题过程

选择一个大于所有 OTP 数字的基数 $m>2^{16}$，再选一个满足最小 32 位检查的公共因子 $c$，提交：

$k_i=c\,m^i,\quad i=0,\ldots,5$。

例如可取 `c = next_prime(2^32)`、`m = next_prime(2^16)`。六个 token 的位数均落在允许的 32 到 1023 位之间。服务泄露的整数为

$L=\sum_{i=0}^{5}k_i\theta_i=c\sum_{i=0}^{5}\theta_im^i$。

由于 $0\leq\theta_i<2^{16}<m$，除以 $c$ 后没有进位冲突，连续取模和整除即可恢复每一位：

```python
value = leaked_inner_product // c
otp = []
for _ in range(6):
    otp.append(value % m)
    value //= m
```

把六个数按原顺序提交，服务返回：

```text
grey{Inner_Product_Funct1onal_3ncrypti0n_pNt8yBfwhz5ZNJzP}
```

题目同时输出的 `Token Hash` 和 `OTP Hash` 对这一版本并非必需；直接的整数内积已经完整泄露 OTP。

## 方法总结

可控权重的精确内积往往能充当编码通道。只要每个秘密分量有已知上界，就可用超出该上界的基数把所有分量无碰撞地打包进一个整数。发布“调试统计量”前必须分析攻击者是否能控制其系数，而不能只看单个秘密是否经过群运算保护。
