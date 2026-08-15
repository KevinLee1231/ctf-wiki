# Parallel CTR PoC

## 题目简述

程序内嵌一个 16 字节 AES 密钥和 80 字节目标密文。用户先提交线程数，再提交候选输入。每个线程只处理一个 16 字节块：线程编号被编码成 128 位大端 counter，使用 AES-128-CTR 对对应输入块做变换，并把结果与内嵌密文同位置比较。

成功条件还要求线程数等于密文块数，因此本题共有 $80/16=5$ 个 counter，依次为 0 到 4。

## 解题过程

CTR 模式用 AES 加密 counter 得到密钥流，再与数据 XOR：

$$
C_i=P_i\oplus E_K(\operatorname{ctr}_i).
$$

XOR 的对称性意味着加密和解密是同一个操作。程序希望“对用户输入做 CTR 变换后等于 enc”，所以正确输入就是用内嵌密钥从 enc 解密出的明文。

uint2block 先把 32 位线程编号写到缓冲区，再反转整个 16 字节数组，效果正好是标准的 128 位大端 counter。可以直接从 counter 0 连续解密：

~~~python
from Crypto.Cipher import AES

KEY = bytes.fromhex(
    "29aeccd20285ee3c843d8d4ecde67984"
)
ENC = bytes([
    # 将源码中的 80 个 enc 字节放在这里
])

cipher = AES.new(
    KEY,
    AES.MODE_CTR,
    nonce=b"",
    initial_value=0,
)
plain = cipher.decrypt(ENC)
print(plain)
~~~

官方 Ruby solver 采用同样的 key 与 OpenSSL AES-128-CTR。实际运行得到：

~~~text
Congrats, you did it! here is your flag: shellmates{encrypt=decrypt_in_ctr_m0de}
~~~

与程序交互时先输入 5，再提交整段恢复文本。官方 solution README 的 flag 行漏写了右花括号；challenge.yml 与实际解密结果都确认完整 flag 为：

~~~text
shellmates{encrypt=decrypt_in_ctr_m0de}
~~~

## 方法总结

逆向 CTR 实现时要确认三件事：密钥是否硬编码、counter 的字节序与初值、各线程处理的块编号。CTR 不需要单独的“解密算法”，同一密钥流 XOR 两次就还原原文。并行化本身不会增加安全性；客户端同时包含 key、ciphertext 和 counter 构造时，静态恢复即可得到候选输入。
