# Game

## 题目简述

题目描述中的 `beast` 是有效提示，本题是 BEAST（Browser Exploit Against SSL/TLS）攻击的简化模拟。服务端生成 48 字节随机 secret，提供的主要能力是把用户可控前缀拼到 secret 前面后用 AES-CBC 加密。漏洞点在于每次 CBC 加密使用的 IV 或后续异或向量可预测，因此攻击者可以控制进入单块 AES 加密前的 16 字节输入。目标是逐字节恢复 48 字节 secret，再提交 secret 换取 flag。

## 解题过程

CBC 加密中，单块进入分组密码前的输入是明文块与前一个密文块（第一块则为 IV）异或后的结果。若记攻击者希望进入 AES 单块加密的输入为 $P$，当前可预测的异或向量为 $iv$，那么提交的明文块可以构造为 $P \oplus iv$。服务端真正送入 AES 的值就是：

$$
(P \oplus iv) \oplus iv = P
$$

因此，虽然服务端只提供 CBC 加密接口，攻击者仍能把其中某一块当作可控输入的 AES-ECB 单块 oracle 使用。AES 单块加密可视为固定密钥下的伪随机置换，同一 16 字节输入会得到同一 16 字节输出。如果已知输入块前 15 字节，只剩最后 1 字节未知，就可以枚举 0 到 255，找到与目标密文块相同的输出，从而恢复该字节。

接下来需要把 secret 的目标字节对齐到块末尾。因为服务端加密的是：

```text
user_controlled_prefix || secret
```

先发送 15 字节填充，就能让第一块变成“15 字节已知填充 + secret[0]”。拿到这一块的密文后，再用可预测 IV 构造 256 个候选输入，比较候选输出是否等于目标密文块。恢复第一个字节后，把它视为已知字节，改为发送 14 字节填充恢复第二个字节；不断重复即可恢复全部 48 字节。

核心枚举逻辑如下，代码中的 `known` 必须是 15 字节，`cipher` 是待匹配的目标密文块，`IV` 是下一次加密前可预测的异或向量：

```python
def xor(a, b):
    return bytes(x ^ y for x, y in zip(a, b))

def get_last_byte(known, cipher, IV, oracle):
    for guess in range(256):
        payload = xor(IV, known + bytes([guess]))
        out = oracle(payload.hex())
        IV = out[-16:]
        if out[:16] == cipher:
            return bytes([guess]), IV
    raise ValueError("byte not found")
```

PDF 中的完整 exploit 还包含 PoW 处理、块边界控制和 4 轮恢复流程。恢复逻辑按 16 字节分块推进：第一轮恢复 `secret[0:15]`，后续轮次借前一密文块作为已知异或向量继续恢复。平均每个字节尝试 128 次，48 字节约需 $128 \times 48 = 6144$ 次查询，题目给的 1200 秒超时足够完成。

最终恢复出的 secret 提交后得到：

```text
WMCTF{Dont_ever_tell_anybody_anything___If_you_do__you_start_missing_everybody}
```

## 方法总结

BEAST 类攻击的识别信号是：CBC 模式、攻击者能控制明文前缀、IV 或前一块密文可预测、目标 secret 被拼接进同一次加密。关键不是“解 AES”，而是利用 CBC 的异或输入性质，把 CBC oracle 转化成可控单块加密 oracle。只要能把未知字节对齐到块尾，并知道同块前 15 字节，就能用 256 次以内的枚举恢复 1 字节 secret。
