# encryption_with_matrices

## 题目简述

题目名称声称使用矩阵，但源码实际对每个字符独立计算同一个数值变换：密文元素为 $1337^2\cdot\operatorname{ord}(c)^2$。字符之间没有线性混合，也不存在真正的矩阵密码结构。

## 解题过程

对每个密文整数 $x$，连续除以两次 1337，再取精确整数平方根即可恢复字符码：

$$
\operatorname{ord}(c)=\sqrt{\frac{x}{1337^2}}.
$$

```python
from math import isqrt

plain = []
for x in encrypted_values:
    q, rem = divmod(x, 1337 * 1337)
    assert rem == 0
    code = isqrt(q)
    assert code * code == q
    plain.append(chr(code))

print("".join(plain))
```

输出为：

```text
n00bz{7h1s_sch3m3_t00k_m3_m0r3_th4n_f1v3_h0urs_t0_m4k3_4nd_5_m1nut3s_t0_s0lv3_xDD}
```

## 方法总结

分析自定义“加密”时应先把源码化为代数表达式，再判断是否真的存在跨字符耦合。本题的变换逐字符、无密钥且完全可逆；精确除法和整数平方根还能提供很强的正确性校验。
