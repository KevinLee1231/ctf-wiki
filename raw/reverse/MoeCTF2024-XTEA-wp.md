# XTEA

## 题目简述

程序要求 12 字节 key，使用 32 轮魔改 XTEA。它先加密字节 `0..7`，把结果写回原缓冲区，再加密此时的字节 `4..11`；两个 8 字节分组重叠四字节，因此解密必须先撤销第二个窗口，再撤销第一个窗口。

## 解题过程

反编译器最初把第三个参数显示成整数，结合访问方式应修正为四个 `uint32_t` 的 key 数组 `(2, 0, 2, 4)`。轮变量每次执行 `sum -= 0x33004445`，等价于以

$$
delta=-0x33004445\pmod{2^{32}}=0xccffbbbb
$$

作为 XTEA delta。

最终 12 字节密文按小端序是三个 dword `(c_0,c_1,c_2)`。第二次加密的输入为“第一次密文的后四字节 + 原明文最后四字节”，所以先解密 `(c_1,c_2)`，得到第一次密文缺失的 dword 和最后一个明文 dword；然后才能解密第一个窗口。

```python
from struct import pack, unpack

MASK = 0xFFFFFFFF
DELTA = (-0x33004445) & MASK
KEY = (2, 0, 2, 4)
ENC = bytes([
    0xA3, 0x69, 0x96, 0x26,
    0xBD, 0x78, 0x0B, 0x3D,
    0x9D, 0xA5, 0x28, 0x62,
])

def xtea_decrypt(v0, v1):
    total = (DELTA * 32) & MASK
    for _ in range(32):
        mix1 = ((((v0 << 4) & MASK) ^ (v0 >> 5)) + v0) & MASK
        mix1 ^= (total + KEY[(total >> 11) & 3]) & MASK
        v1 = (v1 - mix1) & MASK

        total = (total - DELTA) & MASK

        mix0 = ((((v1 << 4) & MASK) ^ (v1 >> 5)) + v1) & MASK
        mix0 ^= (total + KEY[total & 3]) & MASK
        v0 = (v0 - mix0) & MASK
    return v0, v1

word0, word1, word2 = unpack("<3I", ENC)

# 撤销后加密的重叠窗口，再撤销先加密的窗口。
word1, word2 = xtea_decrypt(word1, word2)
word0, word1 = xtea_decrypt(word0, word1)

answer = pack("<3I", word0, word1, word2)
print(answer)
```

输出：

```text
moectf2024!!
```

因此 flag 为：

```text
moectf{moectf2024!!}
```

## 方法总结

识别 XTEA 后仍要还原调用层的数据流。重叠分组会让第二次加密依赖第一次的中间密文，逆向顺序必须与加密调用顺序相反；若直接把 12 字节拆成两个普通独立块，永远无法得到正确 key。
