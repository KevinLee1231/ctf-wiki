# MiniLCTF 2021 - standardcbc

## 题目简述

服务端为每个连接随机生成 16 字节 SM4 密钥和 16～31 字节 `secret`，提供三个接口：

- 加密 `message || secret`，返回 `IV || SM4-CBC(...)` 的 Base64；
- 解密用户给出的密文和 IV，但只返回“解密结果做 Base64 后的最后一个字符”；
- 提交完整 `secret`，正确时返回 flag。

漏洞不是传统的布尔型 PKCS#7 padding oracle。真正泄漏的是 Base64 尾字符；结合 CBC 中“前一密文块可控地异或下一明文块”的性质，可以逐次恢复被对齐到块尾的 secret 字节。

## 解题过程

先利用加密接口确定 `secret` 的长度。不断提交长度为 1～15 的零字节消息，观察返回密文总长度何时增加一个分组。返回值还包含 16 字节 IV，因此可由跃迁位置反推出 secret 的实际长度。

随后为每个目标字节选择合适长度的前缀，使它落在某个明文块末尾。加密接口给出对应的相邻密文块 $C_{i-1},C_i$。CBC 解密满足

$$
P_i=D_K(C_i)\oplus C_{i-1}.
$$

解密接口允许我们自选新的前一块 $C'_{i-1}$。枚举该块最后一个字节，就能控制 $P'_i$ 的末字节。服务端返回的 Base64 最后字符只与末尾少量比特有关；把目标响应调到已知的尾字符后，可由

$$
P_i[-1]=P'_i[-1]\oplus C'_{i-1}[-1]\oplus C_{i-1}[-1]
$$

还原原字节。对不同前缀长度重复该过程即可拼出 secret。下面保留比赛题解使用的核心实现；交互提示可能因部署包装不同而需要调整 `recvuntil`。

```python
from base64 import b64decode, b64encode
from pwn import remote

io = remote("127.0.0.1", 10001)

def encrypt(msg):
    io.sendlineafter(b"3.getflag;", b"1")
    io.sendlineafter(b"your message:", b64encode(msg))
    return b64decode(io.recvline().strip())

def tail_oracle(ciphertext, iv):
    io.sendlineafter(b"3.getflag;", b"2")
    io.sendlineafter(b"your ciphertext:", b64encode(ciphertext))
    io.sendlineafter(b"your iv:", b64encode(iv))
    return io.recvline().strip()

def recover_last_byte(target_block, original_prev_byte):
    # 240 字节会令 Base64 的最后一个可见字符只受末字节低位影响；
    # 比赛环境中以空行作为命中条件。
    prefix = b"B" * 239
    for guess in range(256):
        crafted = prefix + bytes([guess]) + target_block
        if tail_oracle(crafted, b"\x00" * 16) == b"":
            return guess ^ original_prev_byte
    raise RuntimeError("oracle miss")

# 对每个对齐位置调用 encrypt()，选取目标 C_i 及原 C_{i-1} 的末字节，
# 再调用 recover_last_byte()。恢复顺序与前缀长度相反，最后按位置拼接。
```

得到完整 `secret` 后，选择菜单 3，并发送 `base64(secret)`，服务端即返回 flag。题目仓库与公开参赛 WP 没有保存当时远程实例的动态 flag，因此这里不伪造具体值。

## 方法总结

处理“只返回 Base64 的一个字符”时，应先分析编码位布局，而不是把它笼统称为 padding oracle。Base64 每个字符承载 6 比特，尾部是否有 `=` 又取决于原文长度模 3；CBC 则让攻击者通过前一密文块控制下一明文块。两种表示层关系叠加后，单字符响应也足以形成可枚举的字节 oracle。复现时要固定同一连接，因为密钥和 secret 都在连接建立时随机生成。
