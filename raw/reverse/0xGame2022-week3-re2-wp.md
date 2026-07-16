# week3re2

## 题目简述

附件是 UPX 加壳的 Windows 可执行文件。脱壳后可以看到程序把密文按两个 `uint32_t` 为一组处理，使用固定常量 $\Delta=\texttt{0x9E3779B9}$、4 个 32 位密钥字和 48 轮 Feistel 更新，结构属于 XTEA。

## 解题过程

先用 `upx -d re2.exe` 脱壳，再在反编译器中定位密文数组、密钥和轮函数。识别 XTEA 的依据不是函数名，而是以下组合特征：两个 32 位状态字交替更新、移位 4/5 位、黄金分割常量 `0x9E3779B9`，以及通过 `sum & 3`、`(sum >> 11) & 3` 选择密钥字。

题目使用 48 轮，不应机械套用常见实现中的 32 轮。解密时要先撤销对 $v_1$ 的更新，再递减轮常量并撤销 $v_0$；每一步都截断到 32 位。完整脚本如下：

```python
from struct import pack

MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9
ROUNDS = 48

KEY = (
    0x01234567,
    0x89ABCDEF,
    0xFEDCBA98,
    0x76543210,
)

ENC = [
    13473614,
    2548965133,
    1615017528,
    1388039615,
    1980322642,
    3526670554,
    3046759222,
    4277028290,
]


def mix(value):
    return ((((value << 4) & MASK) ^ (value >> 5)) + value) & MASK


def decrypt_block(v0, v1):
    total = (DELTA * ROUNDS) & MASK
    for _ in range(ROUNDS):
        k1 = (total + KEY[(total >> 11) & 3]) & MASK
        v1 = (v1 - (mix(v0) ^ k1)) & MASK

        total = (total - DELTA) & MASK
        k0 = (total + KEY[total & 3]) & MASK
        v0 = (v0 - (mix(v1) ^ k0)) & MASK
    return v0, v1


plain = b"".join(
    pack("<II", *decrypt_block(ENC[i], ENC[i + 1]))
    for i in range(0, len(ENC), 2)
)
print(plain.decode())
```

输出为：

```text
flag{could_you_decrypt_the_xtea}
```

## 方法总结

本题流程是脱壳、识别 XTEA 变种、按反向轮序恢复明文。容易踩坑的地方有三个：轮数是 48 而非常见的 32；Python 整数不会自动溢出，必须用 `& 0xFFFFFFFF` 模拟 `uint32_t`；解出的两个整数要按程序所在平台的内存布局以小端序拼回字节。UPX 下载地址不影响题目机制，故不保留原外链。
