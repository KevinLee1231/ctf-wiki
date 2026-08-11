# Feedback

## 题目简述

服务端每次连接随机生成 AES 密钥和 IV，允许三次任意密文解密，然后使用同一组密钥与 IV，以 CFB-128 加密固定的 48 字节消息。CFB 解密 oracle 会泄露块密码输出；虽然目标密文要等查询结束后才给出，但固定明文允许跨三次连接逐块恢复消息。

## 解题过程

CFB-128 的前三个加密块满足：

$$
C_1=P_1\oplus E_K(IV),
$$

$$
C_2=P_2\oplus E_K(C_1),
$$

$$
C_3=P_3\oplus E_K(C_2).
$$

若向解密 oracle 提交两个块 $X\parallel0^{16}$，返回值为：

$$
Q_1=X\oplus E_K(IV),\qquad Q_2=E_K(X).
$$

所以一次查询既能由 $Q_1\oplus X$ 得到 $E_K(IV)$，也能直接从第二块得到 $E_K(X)$。

第一条连接只查询一个全零块。解密结果就是 $E_K(IV)$；结束查询、收到目标密文后计算：

```text
P1 = C1 XOR E(IV)
```

得到固定第一块：

```text
FLAG is hgame{51
```

第二条连接中，先查询全零块取得本连接的 $E_K(IV)$。因为 $P_1$ 已知，可以在目标密文公布前算出本连接的 $C_1=P_1\oplus E_K(IV)$。再查询 $C_1\parallel0^{16}$，其第二个解密块就是 $E_K(C_1)$。结束查询后即可恢复：

```text
P2 = C2 XOR E(C1)
P2 = b72d4cd23b2fe672
```

第三条连接重复上述过程，并利用已知 $P_2$ 继续预测本连接的 $C_2=P_2\oplus E_K(C_1)$。第三次查询 $C_2\parallel0^{16}$，取得 $E_K(C_2)$，最终恢复：

```text
P3 = C3 XOR E(C2)
P3 = a874cb44020868}.
```

核心流程可以写成：

```python
ZERO = b"\x00" * 16

# 每个 recover_round 都新建连接；oracle(x) 返回 CFB 解密结果，
# finish() 结束剩余查询并返回该连接的三块目标密文。
def recover_round(known_blocks):
    stream_iv = oracle(ZERO)[:16]       # D_CFB(0) = E_K(IV)
    streams = [stream_iv]
    predicted_cipher = []

    for plain in known_blocks:
        cipher = xor(plain, streams[-1])
        predicted_cipher.append(cipher)
        response = oracle(cipher + ZERO)
        streams.append(response[16:32]) # 第二块 = E_K(cipher)

    target = finish()
    index = len(known_blocks)
    block = target[16*index:16*(index+1)]
    return xor(block, streams[index])

p1 = recover_round([])
p2 = recover_round([p1])
p3 = recover_round([p1, p2])
print(p1 + p2 + p3)
```

完整消息为：

```text
FLAG is hgame{51b72d4cd23b2fe672a874cb44020868}.
```

因此 flag 是：

```text
hgame{51b72d4cd23b2fe672a874cb44020868}
```

服务端代码与三轮结果由 [HGAME2020 Crypto Writeup](https://blog.soreatu.com/posts/writeup-for-crypto-problems-in-hgame-2020/) 补齐；正文已解释为何随机密钥和 IV 不能阻止这种跨连接逐块恢复。

## 方法总结

- 核心漏洞：同一连接中先开放 CFB 解密 oracle，再用同一密钥和 IV 加密秘密消息。
- 关键技巧：固定明文让上一轮恢复的块可以预测下一条连接中的当前密文块，从而在目标密文公布前查询所需的 $E_K(C_i)$。
- 修复方向：使用带认证的 AEAD 模式、隔离 oracle 与秘密加密的密钥/nonce，并禁止攻击者对同一上下文提交任意密文。
