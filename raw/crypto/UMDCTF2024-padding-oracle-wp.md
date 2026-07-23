# padding-oracle

## 题目简述

题目给出 AES-128-CBC 加密后的 flag 密文和固定 IV，并提供一个 PKCS#7 填充判定接口。接口只接收两个 AES 分组，即 32 字节密文；解密后若末尾填充合法就返回 `valid padding :)`，否则返回 `invalid padding :(`。

CBC 解密满足

$$
P_i=D_K(C_i)\oplus C_{i-1},
$$

其中第一块使用 IV 代替 $C_{i-1}$。攻击者虽然不知道密钥，却能修改前一密文块并观察填充是否合法，因此可以逐字节恢复中间值
$I_i=D_K(C_i)$ 和明文 $P_i$。

## 解题过程

### 单块攻击

固定目标块 $C_i$，向服务发送伪造前块 $C'_{i-1}$ 与 $C_i$。从最后一个字节向前恢复：当已经求出 $I_i$ 的末尾 $r-1$ 字节时，把伪造块的相应后缀设为

$$
C'_{i-1}[j]=I_i[j]\oplus r,
$$

使解密后的末尾强制变成 $r$ 个值为 $r$ 的字节。再枚举当前字节
$C'_{i-1}[16-r]$，一旦 oracle 返回合法填充，就有

$$
I_i[16-r]=C'_{i-1}[16-r]\oplus r.
$$

最后与原始前块异或即可得到明文字节：

$$
P_i[16-r]=I_i[16-r]\oplus C_{i-1}[16-r].
$$

官方脚本的核心逻辑为：

```python
for recovered in range(16):
    pad_value = recovered + 1
    pos = 15 - recovered

    for guess in range(256):
        prefix = get_random_bytes(pos)
        suffix = xor(intermediate_suffix, bytes([pad_value]) * recovered)
        crafted_prev = prefix + bytes([guess]) + suffix

        if oracle(crafted_prev + target_block):
            intermediate_byte = guess ^ pad_value
            plaintext_byte = original_prev[pos] ^ intermediate_byte
            break
```

首个字节恢复时可能遇到“原密文本来就形成合法填充”的假阳性。更稳健的实现应在命中后再翻转伪造块倒数第二字节复查，确认返回不是由更长的原填充造成。

### 遍历全部密文块

对每个密文块都只发送“伪造前块 + 当前目标块”，恰好满足服务要求的 32 字节长度。第一块以题目给出的 IV 作为原始前块，其余块使用实际的前一密文块：

```python
plaintext = b""
blocks = [ciphertext[i:i + 16] for i in range(0, len(ciphertext), 16)]

for index, block in enumerate(blocks):
    previous = iv if index == 0 else blocks[index - 1]
    plaintext += attack_block(previous, block)

plaintext = unpad(plaintext, 16)
```

恢复并去除 PKCS#7 填充后得到：

```text
UMDCTF{I_l0vE_p@dINg_0rAClE_@tTacKS}
```

## 方法总结

- 核心技巧：利用 CBC 中“前一密文块控制下一明文块”的性质和可观察的 PKCS#7 合法性差异，逐字节恢复中间值。
- 识别信号：服务允许提交密文，并以不同响应暴露解密后的填充是否合法，即使不返回明文，也已经构成 padding oracle。
- 复用要点：攻击每个块只需要该块和一个可控前块；实现时要处理最后一字节的假阳性、完整填充块以及网络重试，否则容易得到局部正确但最终无法解码的结果。
