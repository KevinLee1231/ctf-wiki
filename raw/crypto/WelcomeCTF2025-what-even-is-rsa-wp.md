# What Even Is RSA

## 题目简述

附件给出传统 RSA 公钥参数 $(n,e)$ 和密文 $c$，其中 $e=65537$。漏洞不在复杂攻击，而在最基本的密钥生成：模数 $n$ 是偶数，因此其中一个所谓“素数”直接等于 2。

## 解题过程

先检查模数奇偶性：

```python
assert n % 2 == 0
p = 2
q = n // 2
```

得到因子后按标准 RSA 过程计算：

$$
\varphi(n)=(p-1)(q-1),\qquad d=e^{-1}\bmod\varphi(n)
$$

再解密 $m=c^d\bmod n$：

```python
from Crypto.Util.number import long_to_bytes

p = 2
q = n // 2
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

输出为：

```text
grey{wHy_d0_7h1S_22222_m3}
```

## 方法总结

- 核心技巧：在尝试高级 RSA 攻击前先检查模数的微小因子与奇偶性。
- 识别信号：题面强调“我挑选了素数”，但公开模数却能被 2 整除。
- 复用要点：RSA 初筛应包含 `gcd`、小素数试除和公开因子库；一旦分解成功，回到标准私钥恢复流程即可。
