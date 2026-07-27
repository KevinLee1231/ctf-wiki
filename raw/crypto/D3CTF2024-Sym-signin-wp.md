# Sym_signin

## 题目简述

题目实现了一个 32 位 SPN 分组结构：每轮依次异或轮密钥、经过 4 bit S 盒层和固定置换层，主体共执行 8192 轮，最后再做一次 S 盒和轮密钥异或。主密钥有效空间只有 24 bit，并通过奇偶校验扩展到 32 bit。

决定解法的异常是轮密钥更新函数

```python
def key_schedule(key):
    return ((key << 31 & 0xffffffff)
            + (key << 30 & 0xffffffff)
            + key) & 0xffffffff
```

只有 4 轮周期。题目同时给出多组精心选择的明密文，这正是滑动攻击常见的识别信号：总轮数很大，但轮函数及轮密钥序列在很短周期后重复。

## 解题过程

### 关键观察

若直接对每个 24 bit 候选密钥执行完整的 8192 轮加密，复杂度约为

$$
2^{24}\times8192\approx 2^{37}
$$

利用 4 轮周期，可在多组明密文中猜测滑动对。对一对候选样本，只需检查两种相对方向，并验证 4 轮关系。官方题解给出的样本规模使最坏复杂度降为

$$
6\times5\times2^{24}\times4\approx2^{30}
$$

其中 `key_ex` 将 24 bit 候选按每 3 bit 增加 1 bit 奇偶校验位，恢复实际参与轮函数的 32 bit 密钥：

```python
def key_ex(num: int) -> int:
    result = 0
    bit_position = 0
    while num > 0:
        original_bits = num & 0b111
        parity_bit = original_bits.bit_count() & 1
        result |= (
            original_bits << (bit_position + 1)
        ) | (parity_bit << bit_position)
        num >>= 3
        bit_position += 4
    return result
```

### 滑动对搜索

密文先去掉末轮的密钥异或和 S 盒，得到进入末轮前的状态。随后验证一方经过 4 轮后能否到达另一方；由于不知道滑动方向，两个方向都要检查。下面是搜索核心，`enc_round` 与 `S_layer_inv` 沿用题目加密器中的轮函数和逆 S 盒定义，并非可脱离附件单独运行的完整脚本：

```python
def encrypt_without_last(message: int, key: int, rounds: int) -> int:
    state = message
    for _ in range(rounds):
        state = enc_round(state, key)
        key = key_schedule(key)
    return state

def find_key(plain, cipher):
    for i in range(len(plain)):
        for j in range(i + 1, len(plain)):
            for raw_key in range(0x1000000):
                key = key_ex(raw_key)

                left = S_layer_inv(cipher[i] ^ key)
                right = S_layer_inv(cipher[j] ^ key)

                if (
                    encrypt_without_last(left, key, 4) == right
                    or encrypt_without_last(right, key, 4) == left
                ):
                    return raw_key
    raise ValueError("key not found")
```

官方数据中搜索得到 24 bit 密钥 `0x8D3C1F`。该值还不是直接解密 flag 的分组密钥；题目用 3 字节大端表示计算 SHA-256，并取摘要最后 6 个十六进制字符作为下一把 24 bit 密钥：

```python
import hashlib

def l6shad(value: int) -> int:
    digest = hashlib.sha256(value.to_bytes(3, "big")).hexdigest()
    return int(digest[-6:], 16)
```

先计算 `l6shad(0x8D3C1F)`，再按 8192 轮逆过程逐块解密 `flag.enc`；每解完一块继续对当前 24 bit 密钥执行 `l6shad`，最后按小端 32 位整数拼回字节串。官方材料没有给出最终 flag 明文，因此这里只保留已确认的密钥和完整恢复路径，不补写未经验证的 flag。

## 方法总结

- 核心技巧：利用短周期轮密钥调度构造滑动攻击，把完整 8192 轮验证缩减为 4 轮关系检查。
- 识别信号：轮数极大但轮函数相同、轮密钥序列周期很短，并提供多组有结构的明密文时，应优先检查 slide pair。
- 复用要点：候选密钥可能还要经过奇偶位扩展或哈希派生；滑动方向未知时必须同时验证两个方向，命中后再用完整明密文或最终解密结果确认。
