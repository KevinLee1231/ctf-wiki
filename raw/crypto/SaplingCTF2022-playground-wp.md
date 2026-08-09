# Playground

## 题目简述

题目实现了一个 32 字节、16 轮的 Feistel 密码。轮函数近似 AES 的单轮运算，但每轮密钥只由右半部对 64 取模后索引一个秘密字节，并把该字节扩展使用。服务提供选择明文加密，因此可以构造滑动对，逐个恢复 64 字节密钥表，再逆转 Feistel 网络解密 flag。

## 解题过程

Feistel 一轮可写成：

$$
(L,R)\longmapsto(R,L\oplus F(R,K[R\bmod64])).
$$

对每个索引 $i$，构造整数值模 64 为 $i$ 的 32 字节基准块并请求加密。若先在本地对基准块执行一轮候选变换，再把结果交给同一个 16 轮加密器，正确候选会与原样本形成相差一轮的滑动对。官方 solver 的核心判断是比较一个输出的右半部与另一个输出的左半部：

~~~python
for index in range(64):
    base = long_to_bytes(index, 32)
    encrypted = oracle(base)

    for candidate in range(256):
        stepped = one_round(base, candidate)
        probe = oracle(stepped)
        if probe[:16] == encrypted[16:]:
            key[index] = candidate
            break
~~~

恢复完整 KEY 后，按轮的逆序执行 Feistel 逆变换。Feistel 的优势是轮函数无需可逆；只需交换左右半部并重新异或同一个 $F$：

~~~python
def decrypt(block):
    left, right = block[:16], block[16:]
    for _ in range(16):
        old_right = left
        idx = bytes_to_long(old_right) % 64
        old_left = xor(right, round_function(old_right, key[idx]))
        left, right = old_left, old_right
    return left + right
~~~

解密并去除题目用于补齐块长的点号后得到：

~~~text
maple{sL1D3_7h3_C1ph3R_D0wn}
~~~

## 方法总结

迭代密码若每轮结构和轮密钥规律重复，就可能出现 slide attack。这里的另一个致命弱点是 64 字节表每次只取一个字节，选择明文可以把索引固定并将候选空间压到 256。分析自制密码时，应同时检查轮间对称性、密钥调度强度和攻击者能否控制进入轮函数的状态。
