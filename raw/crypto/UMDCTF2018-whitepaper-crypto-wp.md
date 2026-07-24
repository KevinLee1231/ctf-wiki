# UMDCTF 2018 - Whitepaper Crypto

## 题目简述

附件是一页 PDF。页面给出 AES-128 最后一轮轮密钥和一个 16 字节密文块，要求从已知轮密钥逆推出主密钥并恢复明文。

## 解题过程

逐项视觉核对 PDF 后，得到：

```text
round key 10 = e0d2337a187abb31487b908c38979663
ciphertext   = 9c0ac1346018e078b268dd5eb6dda07e
```

AES-128 的密钥扩展会从 16 字节主密钥生成 44 个 32 位字。对每个 $i\ge4$：

$$
w_i=w_{i-4}\oplus g(w_{i-1})
$$

其中当 $i\bmod4\ne0$ 时，$g$ 只是恒等变换；当 $i\bmod4=0$ 时，还要执行 `RotWord`、`SubWord` 和轮常量异或。由于异或可逆，可从 $w_{40}\ldots w_{43}$ 反向计算到 $w_0\ldots w_3$：

```python
for i in range(43, 3, -1):
    temp = words[i - 1]
    if i % 4 == 0:
        temp = sub_word(rot_word(temp))
        temp[0] ^= rcon[i // 4]
    words[i - 4] = xor_words(words[i], temp)
```

恢复出的 AES 主密钥为：

```text
140f33326e6602306438766e61611631
```

用它对密文做 AES-128 单块解密，明文是：

```text
ter.ps/aeslovers
```

该短地址历史上跳转到一段 Base64 文本；解码结果为：

```text
UMDCTF-{YaY_FoR_UNdeRstand1ng_435}
```

本文已记录外链承载的关键文本及最终结果，因此解题不依赖短链继续可用。

## 方法总结

AES 密钥扩展不是单向函数，知道任意完整的 128 位轮密钥就能反推出主密钥。处理 PDF 密码题时还应逐字符核对十六进制数据；一个字符转写错误就会使整块解密结果失去可读性。
