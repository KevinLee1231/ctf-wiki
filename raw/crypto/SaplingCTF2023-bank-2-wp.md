# bank-2

## 题目简述

服务实现了带自定义前缀填充的 CBC 加密，并在处理攻击者提交的密文时返回三类可区分结果。不同响应暴露了填充是否正确，因此可构造 padding oracle，在不知道 AES 密钥的情况下恢复邮件内容。

## 解题过程

对 CBC 的第 $i$ 个明文块，有：

$$
P_i=D_K(C_i)\oplus C_{i-1}
$$

先固定目标密文块 $C_i$，逐字节修改前一块。为了恢复最后一个字节，枚举修改值直到服务进入“填充正确但业务检查失败”的响应；此时可求出中间值 $I_i=D_K(C_i)$ 的对应字节。之后把已恢复位置统一调整为期望填充值 2、3、……，继续向前恢复。

核心循环可以写成：

~~~python
for pad in range(1, 17):
    pos = 16 - pad
    for guess in range(256):
        forged = bytearray(previous)
        for j in range(pos + 1, 16):
            forged[j] = intermediate[j] ^ pad
        forged[pos] = guess
        if padding_is_valid(bytes(forged) + current):
            intermediate[pos] = guess ^ pad
            plaintext[pos] = intermediate[pos] ^ previous[pos]
            break
~~~

题目在消息开头还有自定义前缀填充，恢复全部块后要按源码规则去掉该前缀，而不能直接套用 PKCS#7。最终邮件内容给出：

~~~text
maple{N07_50_5ECReT_5eCreT2}
~~~

## 方法总结

Padding oracle 的成立条件是攻击者能修改 CBC 前一块，并能区分“解密或填充失败”与“填充成功但后续处理失败”。实际利用时应先用几个探针确认每类响应含义，再逐字节恢复中间值；最后还要区分协议自定义填充与标准 PKCS#7，避免得到看似可读却错位的明文。
