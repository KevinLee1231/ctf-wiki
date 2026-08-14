# Recursion

## 题目简述

WelcomeCTF2021 的 Recursion 是一个多层自解密 ELF。程序要求一个长度为 4 的倍数的参数，取前四字节作为种子，解密内置数据到匿名 `memfd`，再通过 `fexecve` 执行新程序并把剩余参数传给下一层。每层结构相同，正确的四字节种子依次组成 flag。

## 解题过程

反编译首层可见，程序把参数前四字节按小端序组成 32 位状态，并逐字节异或内置数组。每处理四字节后更新状态：

$$
x_{i+1}=x_i(x_i+\texttt{0xdeadbeef})\pmod{2^{32}}.
$$

解密后数据被写入 `memfd_create` 返回的文件描述符，随后 `fexecve` 执行；下一层收到的是原参数去掉前四字节后的部分。

每一层的明文都应是 ELF，所以前四个明文字节已知为 `7f 45 4c 46`。设密文数组前四字节为 $c_0\ldots c_3$，初始种子字节就是：

$$
k_i=c_i\oplus \text{ELFMagic}_i.
$$

按小端序合成种子后即可解完整层：

```python
ELF_MAGIC = b"\x7fELF"

def recover_layer(ciphertext):
    seed_bytes = bytes(c ^ m for c, m in zip(ciphertext[:4], ELF_MAGIC))
    state = int.from_bytes(seed_bytes, "little")
    plaintext = bytearray()

    for offset, value in enumerate(ciphertext):
        if offset and offset % 4 == 0:
            state = state * (state + 0xdeadbeef) & 0xffffffff
        key_byte = state.to_bytes(4, "little")[offset % 4]
        plaintext.append(value ^ key_byte)

    assert plaintext.startswith(ELF_MAGIC)
    return seed_bytes, bytes(plaintext)
```

从首层提取内置加密数组，运行函数得到第一段 `grey` 和下一层 ELF。对新 ELF 重复定位数组、恢复种子与解密，共处理八层。也可以用 `strace` 观察每层 `write`，以新增的 ELF 写入和魔数作为候选前缀的判定 oracle。

各层四字节依次拼接，得到：

```text
greyhats{p4cK_@ll_th3_th1Ng5!!!}
```

最后一层不再生成下一层，只输出结束信息。

## 方法总结

题名中的“递归”指相同解密器层层生成并执行下一份 ELF。突破点是每层明文都具有固定 ELF 魔数，因此四字节密钥无需爆破。每次解层后应验证 ELF 头和长度，再继续分析，避免把错误密钥传播到后续层。
