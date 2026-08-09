# RSA

## 题目简述

服务每次生成新的 RSA 模数，但始终用小指数 $e=17$ 加密同一条、未填充的消息。收集 17 组密文和模数后，可以执行 Håstad 广播攻击。

## 解题过程

连接服务 17 次，记录 $(c_i,n_i)$。在模数两两互素且消息满足广播攻击范围时，中国剩余定理可恢复：

$$C\equiv m^{17}\pmod{\prod_{i=1}^{17}n_i}.$$

由于乘积模数足够大，CRT 结果就是整数意义下的 $m^{17}$，再取精确 17 次整数根：

```python
from Crypto.Util.number import long_to_bytes
from gmpy2 import iroot

C = crt(ciphertexts, moduli)
m, exact = iroot(C, 17)
assert exact
print(long_to_bytes(int(m)))
```

恢复得到：

```text
n00bz{5m4ll_3_1s_n3v3r_g00d!}
```

## 方法总结

小指数本身不是问题，问题是对同一消息在多个互素模数下进行确定性、无填充加密。实际系统应使用带随机性的 RSA-OAEP，而不是直接计算 $m^e\bmod n$。
