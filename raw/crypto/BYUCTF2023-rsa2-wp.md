# BYUCTF 2023 - RSA2

## 题目简述

附件中的 RSA 模数由足够大的随机素数生成，直接分解并不现实；但出题人预先把这个具体模数提交到了公开因数数据库，破坏了素因子的保密性。

## 解题过程

把附件中的 `n` 提交到 [FactorDB](https://factordb.com/)，数据库直接返回两个完整素因子 `p` 和 `q`。拿到因子后无需攻击 RSA 算法本身：

```python
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(m.to_bytes((m.bit_length() + 7) // 8, 'big'))
```

结果为：

```text
byuctf{rsa_is_only_secure_when_p_and_q_are_unknown}
```

## 方法总结

大模数不自动意味着安全；若因子曾被公开、复用或泄漏，标准解密即可完成。RSA 题的初查应包括 FactorDB、素数复用和生成日志，而不是一开始就尝试昂贵的通用分解。
