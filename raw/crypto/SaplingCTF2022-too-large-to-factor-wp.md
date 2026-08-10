# Too Large to Factor

## 题目简述

题目使用 1024 位 RSA 模数和很小的公共指数 $e=3$。flag 转成的整数不足 256 位，又没有 OAEP 或随机填充，因此 $m^3<N$。模运算实际上没有发生回绕，密文就是普通整数立方 $c=m^3$，无需分解看似很大的 $N$。

## 解题过程

先比较数量级：若 $m<2^{256}$，则 $m^3<2^{768}$，而 $N$ 约为 1024 位，所以必有 $m^3<N$。RSA 公式：

$$
c=m^3\bmod N
$$

退化为：

$$
c=m^3.
$$

求精确整数立方根并检查余数为零：

~~~python
from Crypto.Util.number import long_to_bytes
from gmpy2 import iroot

root, exact = iroot(ciphertext, 3)
assert exact
assert pow(root, 3) == ciphertext
print(long_to_bytes(int(root)))
~~~

输出为：

~~~text
maple{d0n7_f0rg37_y0ur_p4dd1n9!}
~~~

这里“模数太大”不是保护，反而让短消息的低指数立方完全落在模数范围内。

## 方法总结

低指数 RSA 必须配合安全的随机填充。看到 $e=3$ 时应立即检查消息上界、密文是否为完全立方，以及多接收者广播攻击等条件。攻击前做位数估算往往比尝试分解模数更重要。
