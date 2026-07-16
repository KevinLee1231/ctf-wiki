# Fault!Fault!

## 题目简述

服务先按 RSA 公钥运算返回 `c=m^e mod n`，故障接口则把私钥指数的第 $i$ 位翻转，即使用 $d_i=d\oplus2^i$ 计算 $m_i=c^{d_i}\bmod n$。选择一个已知且与 $n$ 互素的消息，逐位比较正常明文与故障结果，就能恢复完整私钥 $d$。

## 解题过程

令真实私钥第 $i$ 位为 $b_i$，正常解密结果为 $m=c^d\bmod n$。

- 若 $b_i=0$，翻转后 $d_i=d+2^i$，所以 $m_i\equiv m\cdot c^{2^i}\pmod n$；
- 若 $b_i=1$，翻转后 $d_i=d-2^i$，所以 $m_i\equiv m\cdot c^{-2^i}\pmod n$。

两边乘 $m^{-1}$，令 $h_i=m_i m^{-1}\bmod n$，便可直接判断：

$$
h_i=
\begin{cases}
c^{2^i}\bmod n,&b_i=0,\\
(c^{2^i})^{-1}\bmod n,&b_i=1.
\end{cases}
$$

先通过 `[S]ign` 发送固定消息 `test`，保存服务返回的 `n`、`e` 和 `c`。随后对每个索引调用 `[F]ault injection`，把故障解密结果按顺序保存。核心恢复代码如下，其中 `faults[i]` 对应第 $i$ 位：

```python
from Crypto.Util.number import bytes_to_long, inverse

m = bytes_to_long(b"test")
inv_m = inverse(m, n)

d = 0
delta = c  # c^(2^0)
for i, faulty_plain in enumerate(faults):
    h = faulty_plain * inv_m % n
    if h == delta:
        bit = 0
    elif h == inverse(delta, n):
        bit = 1
    else:
        raise ValueError(f"unexpected oracle result at bit {i}")

    d |= bit << i
    delta = pow(delta, 2, n)  # c^(2^(i+1))

assert pow(c, d, n) == m
print(d)
```

查询数量取 `n.bit_length()`，逐位从 0 开始；恢复后必须用 `pow(c,d,n)==m` 校验，再通过 `[C]heck answer` 提交 $d$。交互服务有 300 秒限制，应先把故障值快速采集到内存或文件，再离线计算，且不要在循环中开启冗长调试日志。

## 方法总结

本题不是传统 CRT-RSA 单次错误签名分解，而是“可控翻转私钥指数位”的故障 oracle。关键是写出 $d\oplus2^i$ 在原位为 0/1 时分别等于加法或减法，再通过已知明文归一化消去 $m$。芯片实现应对私钥运算做完整性校验、故障检测和访问频率限制。
