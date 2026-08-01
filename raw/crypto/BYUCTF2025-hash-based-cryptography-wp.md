# Hash Based Cryptography

## 题目简述

服务把 flag 同时作为密钥和待加密消息。密钥流以 `KEY` 为初始状态，每 20 字节计算一次 SHA-1：$S_1=\operatorname{SHA1}(KEY)$、$S_{i+1}=\operatorname{SHA1}(S_i)$，再与按 20 字节补齐的消息异或。服务允许提交任意十六进制密文解密，并把 `Padding error` 与普通解密失败区分开。

其 `unpad` 只检查末字节 $p\le20$ 且 `message[-p] == p`，没有验证中间所有填充字节，形成了可查询的弱填充 oracle。

## 解题过程

对长度为 20 的全零密文，解密前的明文恰好是首个密钥流块 $S_1$。虽然服务不直接返回它，但可以从末尾向前逐字节构造候选：假设目标填充值为 $j$，把已恢复的尾部字节调整为 $j$，枚举当前密文字节；只要响应不含 `Padding error`，就满足了 `message[-j] == j`，从而恢复当前位置的密钥流字节。

下面是逐块恢复的核心骨架；`oracle` 需要根据响应中是否出现 `Padding error` 返回布尔值：

```python
def recover_block(oracle, prefix_len=0):
    known = b""
    for j in range(1, 21):
        for guess in range(256):
            payload = b"\x00" * prefix_len
            payload += b"\x00" * (20 - j)
            payload += bytes([guess])
            payload += bytes(x ^ j for x in known)
            if oracle(payload.hex()):
                known = bytes([guess ^ j]) + known
                break
        else:
            raise RuntimeError("no candidate")
    return known
```

服务还错误地允许末字节为 0，因此第一个字节的枚举可能出现假阳性。对 $j=1$ 的候选，可逐个翻转同一块中其余 19 个字节并重查：真正令末字节解密为 1 的候选始终通过，因为它只与自身比较；因某个 $p>1$ 偶然通过的候选会在翻转 `-p` 位置时失败。

依次把提交长度扩展为 20、40、60 字节，并把前面块作为占位，可恢复 $S_1,S_2,\ldots$。将这些块拼成密钥流，与服务开场给出的加密 flag 异或即可。官方脚本把已经手工恢复的一块写死在代码里；上述过程解释了该常量的来源，也能完整自动化。

最终得到：

```text
byuctf{my_k3y_4nd_m3ss4g3_w3r3_th3_s4m3}
```

## 方法总结

- 核心技巧：利用可区分的弱填充错误，把解密接口变成逐字节密钥流 oracle，再直接异或目标密文。
- 识别信号：流式异或加密本身不提供完整性；若解密端还暴露不同错误，哪怕填充检查并非标准 PKCS#7，也可能泄露明文或密钥流。
- 复用要点：以服务端真实判断条件推导 payload，不要机械套用 CBC padding oracle；还要处理假阳性并用额外查询验证候选。
