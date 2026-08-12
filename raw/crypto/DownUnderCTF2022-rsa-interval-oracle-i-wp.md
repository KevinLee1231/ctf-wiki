# DownUnderCTF 2022 rsa interval oracle i Writeup

## 题目简述

服务生成 384 位 RSA 模数 $N$，加密一个随机非零秘密 $m$ 并公开 $c=m^e\bmod N$。玩家最多可以添加 384 个开区间，并提交总计 384 个密文；oracle 返回解密结果落入的第一个区间编号。猜中 $m$ 即可获得 flag。

第一题允许反复改变区间，因此原始密文自身就能作为比较对象，不需要利用 RSA 的乘法性质。

## 解题过程

维护秘密的候选闭区间 $[L,H]$，初始为 $[1,N-1]$。每轮取中点 $M$，添加区间 $(0,M)$ 并查询原始密文 $c$：

- 返回该区间编号，说明 $0<m<M$，令 $H=M-1$；
- 返回 `-1`，说明 $m\ge M$，令 $L=M$。

```python
low, high = 1, N - 1
while low < high:
    mid = (low + high + 1) // 2
    if decrypts_into_interval(c, 0, mid):
        high = mid - 1
    else:
        low = mid

assert pow(low, e, N) == c
submit(low)
```

每次把搜索空间缩小约一半，384 位空间最多需要 384 次比较，刚好满足查询和区间数量限制。提交恢复出的整数后得到：

```text
DUCTF{d1d_y0u_us3_b1n4ry_s34rch?}
```

## 方法总结

只要解密 oracle 允许攻击者自由指定区间，它就等价于对明文提供比较操作。面对这类接口应先检查能否直接对目标密文二分，而不是一开始就套复杂的 RSA padding attack。实现时必须留意区间是开区间还是闭区间，并用 `pow(candidate, e, N) == c` 消除最后的边界歧义。
