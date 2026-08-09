# Signature Showdown

## 题目简述

服务使用未哈希、未填充的 textbook RSA 签名：

$$
s=m^d\bmod N
$$

这种签名具有乘法同态。服务拒绝直接签署目标消息，但允许签署其他整数，因此可以把目标分解为两个可签消息的乘积，再组合签名。

## 解题过程

设目标消息为 $m_t$，已知一个可用消息 $m_1$ 及其签名 $s_1$。构造：

$$
m_2=m_t\cdot m_1^{-1}\pmod N
$$

向服务请求 $m_2$ 的签名 $s_2$，再计算：

$$
s_t=s_1s_2\bmod N
$$

验证时：

$$
s_t^e\equiv m_1m_2\equiv m_t\pmod N
$$

实现时用 Python 3 的 pow(m1, -1, N) 求逆：

~~~python
m2 = target * pow(m1, -1, N) % N
s2 = request_signature(m2)
forged = s1 * s2 % N
submit(forged)
~~~

服务接受伪造签名并返回：

~~~text
maple{b3_c4r3ful_w1th_s1gnatur3s!!}
~~~

## 方法总结

RSA 原始幂运算不是安全签名方案，因为乘法同态允许组合合法签名。实际系统需要先哈希并采用 RSASSA-PSS 或严格实现的 PKCS#1 v1.5 编码。本题脚本中最容易出错的是把整数与字节串转换混在一起，或忘记所有运算都在模 $N$ 下进行。
