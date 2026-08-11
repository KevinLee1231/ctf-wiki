# LikiPrime

## 题目简述

题目使用标准 RSA，但两个素因子不是随机生成，而是从固定指数列表中取梅森素数：$p=2^u-1$、$q=2^v-1$。候选指数只有 `1279、2203、2281、3217、4253、4423`，搜索空间极小，因此无需通用大整数分解，逐个枚举候选因子即可恢复私钥。

## 解题过程

生成素数的函数从 1 开始左移 `secret` 次，再减 1：

```python
def get_prime(secret):
    prime = 1
    for _ in range(secret):
        prime <<= 1
    return prime - 1
```

也就是直接计算 $2^{secret}-1$。题目已经限定了候选指数：

```python
secrets = [1279, 2203, 2281, 3217, 4253, 4423]
```

对每个候选 $r=2^k-1$ 检查 $n\bmod r$。一旦余数为 0，便得到 $p=r$ 与 $q=n/p$；随后按普通 RSA 计算

$$
\varphi(n)=(p-1)(q-1),\qquad d\equiv e^{-1}\pmod{\varphi(n)},\qquad m\equiv c^d\pmod n.
$$

官方 PDF 的代码块把具体 `n` 与 `c` 留空，因此下面的完整脚本改为从标准输入读取附件给出的两个整数：

```python
from Crypto.Util.number import long_to_bytes


def mersenne_prime(exponent):
    return (1 << exponent) - 1


exponents = [1279, 2203, 2281, 3217, 4253, 4423]
n = int(input("n = ").strip())
c = int(input("c = ").strip())
e = 0x10001

for exponent in exponents:
    candidate = mersenne_prime(exponent)
    if n % candidate == 0:
        p = candidate
        q = n // p
        break
else:
    raise ValueError("n 不含给定列表中的梅森素因子")

phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

同期选手的 [HGAME 第三、四周 Crypto 复盘](https://huangx607087.online/2021/02/28/HgameDiv3/) 保存了 PDF 缺失的运行结果：

```text
hgame{Mers3nne~Pr!Me^re4l1y_s0+5O-li7tle!}
```

## 方法总结

RSA 模数很大并不等于难以分解，关键取决于因子的生成方式。这里两个因子都来自仅六个元素的梅森素数候选集，把分解问题降成了六次整除测试。看到 `1 << k` 再减 1 时，应立即识别 $2^k-1$ 结构，并结合候选范围判断能否枚举。外部复盘只补足官方 PDF 丢失的最终输出，算法与求解步骤均已在正文展开。
