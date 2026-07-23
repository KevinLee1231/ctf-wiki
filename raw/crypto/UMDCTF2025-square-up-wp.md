# UMDCTF 2025 - square-up

## 题目简述

题目使用 Rabin 加密：选择满足 $p\equiv q\equiv3\pmod4$ 的两个 384 位素数，令 $N=pq$，密文为 $c=m^2\bmod N$。为从四个平方根中选回原消息，程序还输出消息在模 $p$、模 $q$ 下的 Legendre 符号。

生成器同时打印了解密函数返回的错误明文。错误不是无用日志：CRT 拼接中的变量写错，使该明文只在一个素数模数下仍是 $c$ 的平方根，从而直接泄露 $N$ 的因子。

## 解题过程

正确解密先计算：

$$
m_p=c^{(p+1)/4}\bmod p,\qquad
m_q=c^{(q+1)/4}\bmod q,
$$

再根据给出的 Legendre 符号分别选择 $m_p$ 或 $p-m_p$、$m_q$ 或 $q-m_q$。源码第二个分支却写成：

```python
if legendre(mq, q) != lq:
    mq = q - mp       # 应为 q - mq
```

设程序打印的错误解密结果为 $r$。CRT 公式仍保证 $r\equiv m_p\pmod p$，所以：

$$
r^2\equiv c\pmod p.
$$

但错误的 $q$ 分量通常不满足平方根关系，因此 $r^2-c$ 只含有素因子 $p$。直接计算：

```python
p = gcd(r * r - c, N)
q = N // p
```

即可分解模数。修正变量名后重新执行 Rabin 解密：

```python
if pow(mp, (p - 1) // 2, p) != lp % p:
    mp = p - mp
if pow(mq, (q - 1) // 2, q) != lq % q:
    mq = q - mq
```

最后按 CRT 合并得到：

```text
UMDCTF{e=3_has_many_attacks_and_e=2_has_its_own_problems...maybe_we_should_try_e=1_next?}
```

## 方法总结

- 核心技巧：利用“部分正确”的错误 CRT 结果，通过 $\gcd(r^2-c,N)$ 提取仍然成立的模因子。
- 识别信号：RSA/Rabin 程序公开了解密失败输出、故障签名或只有一个 CRT 分支被污染的结果。
- 复用要点：面对错误明文不要只尝试修代码；先检查它与密文之间的代数关系。只在 $p$ 或 $q$ 一侧成立的等式往往就是故障攻击的分解入口。
