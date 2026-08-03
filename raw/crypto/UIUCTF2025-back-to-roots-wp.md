# back to roots

## 题目简述

题目随机选择整数 $K\in[10^{10},10^{11})$，用 Python `Decimal` 计算 $\sqrt K$，但只泄漏十进制小数点后的数字：

```python
K = randint(10**10, 10**11)
leak = int(str(Decimal(K).sqrt()).split('.')[-1])
```

随后以十进制字符串 `str(K)` 的 MD5 摘要作为 AES-128 密钥，用 ECB 模式和 PKCS#7 padding 加密 flag。关键问题不是求一般意义上的平方根逆像，而是 `Decimal` 默认只有 28 位有效数字；$K$ 的取值范围又把 $\sqrt K$ 的整数部分限制在很小的区间内。

## 解题过程

由 $10^{10}\le K<10^{11}$ 可知：

$$
100000\le\lfloor\sqrt K\rfloor\le316227.
$$

本题泄漏值有 22 位，和 6 位整数部分合起来正好对应 28 位十进制有效数字。枚举整数部分 $n$，构造：

$$
x_n=n+\frac{\text{leak}}{10^{22}},
$$

再检查 $x_n^2$ 与最近整数的距离。正确整数部分产生的平方会极接近某个整数 $K$，错误候选通常没有这种性质。计算时需要使用高精度浮点数，不能退回普通 `float`。

```python
import mpmath
from hashlib import md5
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

leak = 4336282047950153046404
ct = bytes.fromhex(
    "7863c63a4bb2c782eb67f32928a1dece"
    "aee0259d096b192976615fba644558b2"
    "ef62e48740f7f28da587846a81697745"
)

mpmath.mp.dps = 40
best = None

for n in range(100000, 316228):
    root = mpmath.mpf(f"{n}.{leak}")
    square = root**2
    candidate = int(mpmath.nint(square))
    error = abs(square - candidate)
    if best is None or error < best[0]:
        best = (error, candidate, n)

error, K, integer_part = best
key = md5(str(K).encode()).digest()
flag = unpad(AES.new(key, AES.MODE_ECB).decrypt(ct), 16)

print(integer_part, K, error)
print(flag)
```

复核得到整数部分 `204064`、$K=41642293072$，平方误差约为 $1.12\times10^{-17}$。解密结果为：

```text
uiuctf{SQu4Re_Ro0T5_AR3nT_R4nD0M}
```

## 方法总结

- 核心技巧：利用高精度十进制平方根泄漏和已知取值范围，枚举短整数部分，并以“平方是否接近整数”作为验证条件。
- 识别信号：秘密整数范围较小、只泄漏无理数近似的小数部分、输出精度固定，同时后续密钥可以验证候选。
- 复用要点：必须复现原程序的有效数字精度；普通二进制浮点误差可能淹没区分信号。范围推导后只枚举 $216228$ 个整数部分，无需照搬更宽的百万级搜索。
