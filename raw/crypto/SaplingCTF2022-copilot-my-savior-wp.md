# Copilot My Savior

## 题目简述

附件把 flag 的每个字节与同一个单字节密钥异或。密钥空间只有 256 种，即使不知道密钥，也可以穷举全部候选并用 flag 格式或可打印字符比例筛选。

## 解题过程

重复密钥异或满足：

$$
c_i=m_i\oplus k,\qquad m_i=c_i\oplus k.
$$

因为 $k$ 只有一个字节，直接遍历 0 至 255：

~~~python
ciphertext = bytes.fromhex(CIPHERTEXT_HEX)

for key in range(256):
    plaintext = bytes(byte ^ key for byte in ciphertext)
    if plaintext.startswith(b"maple{") and plaintext.endswith(b"}"):
        print(key, plaintext.decode())
~~~

如果不知道固定前缀，也可以按可打印 ASCII 比例、英语频率或括号结构打分。唯一合理候选为：

~~~text
maple{c0P1L07_b37R4Y3D_m3}
~~~

## 方法总结

异或本身没有提供安全性；安全性来自与消息等长、真正随机且只使用一次的密钥流。单字节密钥只有 8 位熵，穷举成本可以忽略。处理这类题时应先估算有效密钥空间，再考虑更复杂的频率分析或已知明文攻击。
