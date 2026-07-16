# Ez_RSA

## 题目简述

附件执行标准 RSA：随机生成两个 256 位素数 $p,q$，令 $n=pq$、$e=65537$，再输出模数 $n$ 与密文 $c=m^e\bmod n$。题目的薄弱点不是 RSA 公式，而是给定模数已经能够被公开数据库分解。

## 解题过程

将题目给出的 $n$ 提交到 [FactorDB](https://factordb.com/)，可以得到两个素因子。FactorDB 在本题中只负责提供已知分解；取得 $p,q$ 后，仍需验证 $pq=n$，再按标准 RSA 私钥推导完成解密：

```python
from Crypto.Util.number import long_to_bytes

n = ...  # 题目输出
c = ...  # 题目输出
e = 65537
p = ...  # FactorDB 返回的素因子
q = ...

assert p * q == n
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

验证乘积能够避免抄错因子；随后利用 $ed\equiv1\pmod{\varphi(n)}$ 求出私钥指数 $d$，计算 $m=c^d\bmod n$ 并转回字节即可。

## 方法总结

- 核心技巧：查询公开整数分解结果，再执行标准 RSA 私钥恢复。
- 识别信号：常规 RSA 参数没有额外结构，但模数规模较小或可能已被收录时，应先检查现成分解结果。
- 复用要点：外部分解服务只替代因数分解步骤，后续仍要验证 $pq=n$ 并自行计算 $\varphi(n)$、$d$ 和明文。
