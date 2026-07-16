# 幻视调律

## 题目简述

附件是一个 64 位 Windows PE，要求输入长度为 44 字节的 flag。程序表面上用固定种子生成 S 盒，再以 RC4 风格的流加密处理输入并调用 `strcmp`；沿这条静态路径反解会得到一个明确写着 `fake_flag` 的诱饵。动态跟踪可发现 `strcmp` 的目标在运行时被替换，真实比较函数还会对输入执行 10 轮重叠分组的 XTEA 风格变换，再与 44 字节常量比较。

## 解题过程

### 识别静态假 flag

程序的第一层算法使用 `srand(0x114514)` 初始化 Windows CRT 伪随机数生成器，再用每次 `rand()` 的低 8 位打乱 256 字节 S 盒。后续 PRGA 与 RC4 类似，但每次调用都把局部索引 `i`、`j` 重新置零，同时沿用上一次调用已经改变的 S 盒。

静态分析得到的两个密文分别为：

```text
s1 = fb10a563a44262ec21fd81fb5728a431c6
s2 = 9fbeaf3f8712f05075c0a4f678427755807b02675efe07ec7677c83166950a257be283029c7b9ce9932e66c7
```

按程序顺序解密 `s1` 和 `s2` 会得到：

```text
Input the flag:
0xGame{Wh4t_@_p1ty!!_It_is_just_a_fake_flag}
```

第二个字符串本身已经提示这是诱饵，说明不能停在静态可见的 `RC4(input); strcmp(input, s2)` 路径。

### 找到被替换的真实比较函数

在 `strcmp` 调用处进入汇编单步跟踪，实际落点不是标准库，而是一段运行时替换的比较函数。该函数把 44 字节输入视为 11 个小端 `uint32_t`，对相邻的两个字重复执行 32 轮 XTEA 风格运算：

```c
for (int i = 0; i < 10; ++i) {
    uint32_t left = input[i];
    uint32_t right = input[i + 1];
    uint32_t sum = 0x9E3779B9;

    for (int round = 0; round < 32; ++round) {
        left += (sum + right) ^ (key[0] + 16 * right)
                ^ ((right >> 5) + key[1]);
        sum += 0x9E3779B9;
        right += (sum + left) ^ (key[2] + 16 * left)
                 ^ ((left >> 5) + key[3]);
    }

    input[i] = left;
    input[i + 1] = right;
}
```

这里 `key` 就是原始 `s2` 的前 16 字节，目标密文则是以下 11 个 32 位整数：

```text
4238dbd9 3ad923e5 c1f2bc39 3a5d3d21 0b4c8bb1 0a5ea10e
40f9bfd1 dcb94bf7 e876f233 27da1aa5 b952bacb
```

每轮处理的是 `(input[i], input[i + 1])`，相邻轮次会共享一个字。因此解密时不能把它当成五个独立的 64 位块，必须先把外层顺序从 `i = 9` 倒推到 `i = 0`，每组内部再逆 32 轮。

### 完整求解

下面的脚本自行实现 MSVCRT 的 `rand()`，从而不依赖运行平台的 C 标准库实现。先解密重叠 XTEA 风格变换，再用处理完提示字符串后的同一 S 盒恢复原始输入：

```python
import struct

MASK = 0xFFFFFFFF

prompt_cipher = bytearray.fromhex("fb10a563a44262ec21fd81fb5728a431c6")
fake_flag_cipher = bytes.fromhex(
    "9fbeaf3f8712f05075c0a4f678427755807b02675efe07ec"
    "7677c83166950a257be283029c7b9ce9932e66c7"
)
target = [
    0x4238DBD9,
    0x3AD923E5,
    0xC1F2BC39,
    0x3A5D3D21,
    0x0B4C8BB1,
    0x0A5EA10E,
    0x40F9BFD1,
    0xDCB94BF7,
    0xE876F233,
    0x27DA1AA5,
    0xB952BACB,
]


class MsvcrtRand:
    def __init__(self, seed):
        self.state = seed

    def rand(self):
        self.state = (self.state * 214013 + 2531011) & MASK
        return (self.state >> 16) & 0x7FFF


rng = MsvcrtRand(0x114514)
sbox = list(range(256))
j = 0
for i in range(256):
    j = (j + sbox[i] + (rng.rand() & 0xFF)) & 0xFF
    sbox[i], sbox[j] = sbox[j], sbox[i]


def rc4_transform(data):
    i = 0
    j = 0
    for offset in range(len(data)):
        i = (i + 1) & 0xFF
        j = (j + sbox[i]) & 0xFF
        sbox[i], sbox[j] = sbox[j], sbox[i]
        data[offset] ^= sbox[(sbox[i] + sbox[j]) & 0xFF]


# 程序先解密提示字符串，因此真实 flag 处理时使用的是已改变的 S 盒。
rc4_transform(prompt_cipher)
assert prompt_cipher == b"Input the flag: \n"

key = struct.unpack("<4I", fake_flag_cipher[:16])
words = target[:]

for i in range(9, -1, -1):
    left = words[i]
    right = words[i + 1]
    total = 0x6526B0D9

    for _ in range(32):
        right = (
            right
            - (
                (total + left)
                ^ (key[2] + 16 * left)
                ^ ((left >> 5) + key[3])
            )
        ) & MASK
        total = (total + 0x61C88647) & MASK
        left = (
            left
            - (
                (total + right)
                ^ (key[0] + 16 * right)
                ^ ((right >> 5) + key[1])
            )
        ) & MASK

    words[i] = left
    words[i + 1] = right

flag = bytearray(struct.pack("<11I", *words))
rc4_transform(flag)
print(flag.decode())
```

脚本输出：

```text
0xGame{20994d4d-30ab-4918-b00a-5ca6ae138613}
```

因此 flag 为 `0xGame{20994d4d-30ab-4918-b00a-5ca6ae138613}`。

## 方法总结

本题用“可完整解出的假 flag”诱导选手过早结束静态分析。以后遇到结果主动声明自己是 fake、程序导入的比较函数行为与返回结果不一致，或调试时调用目标跳入非模块代码区，应继续核对运行时调用目标。算法逆向时还要特别留意状态复用和重叠分组：本题的 S 盒会跨调用保留，而 11 个字组成的 10 个相邻字对必须逆序解密，忽略任一细节都会得到看似合理但错误的输出。
