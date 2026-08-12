# 神秘代码 2

## 题目简述

每一问都会给出两行数据：第一行是 Base64 编码的 Uxn ROM 字节码，第二行是该 ROM 的运行结果。ROM 的标准输入不是明文 flag，而是 `zlib.compress(flag)` 的结果，因此完整逆向链是“识别 ROM 的编码逻辑 → 还原输出字节 → zlib 解压”。

前两问围绕变形 Base64 展开，第三问则把输入位重排为四个 6 位组，再拆成半字节并叠加由命令行参数提供的循环密钥。题目中虽然出现 Base64、Vigenère 式加法和压缩，但必须先恢复 Uxn/Varvara 字节码的控制流与数据流，决定性主障碍是自定义虚拟机逆向，因此归入 `reverse`。

## 解题过程

### 识别 Uxn ROM

对第一行做 Base64 解码后可以得到 ROM。Uxn 程序通常从 `0x0100` 开始执行，并通过 Varvara 设备页完成输入输出；本题频繁访问的 Console 字段是：

- `0x10`：Console/vector，写入事件处理函数地址；
- `0x12`：Console/read，读取当前输入字节；
- `0x17`：Console/type，判断普通输入或参数结束；
- `0x18`：Console/write，输出一个字节。

可用 [`uxndis`](https://git.sr.ht/~rabbits/uxn-utils/tree/main/item/cli/uxndis) 得到基础反汇编，再依据 `BRK`、`JMP2r` 和写入 Console/vector 的地址手工划分事件处理器与子函数。阅读指令时还要注意 Uxn 的工作栈、返回栈以及 8/16 位指令变体；外部的 [Uxntal 指令表](https://wiki.xxiivv.com/site/uxntal_opcodes.html) 只用于核对语义，下面已把本题真正用到的数据变换完整写出。

### 第一、二问：从 ROM 尾部恢复 Base64 字母表

第一问反汇编后可以识别出典型的三字节转四个 6 位索引、结尾补 `=` 的 Base64 流程。ROM 末尾紧跟一张 64 字节表，但它不是标准的
`ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/`，运行结果因此看起来像另一层密文。

第二问在生成索引和输出字符之间加入了仿射运算及递增状态，表面上不再是直接查表。不过附件 `alterb64.tal` 的数据流表明，这些运算与精心构造的 `mime` 表相互抵消，最终输出仍可按 ROM 最后 64 字节在表中的位置映射回标准 Base64 索引。两问可以使用同一还原脚本：

```python
import base64
import string
import zlib

rom_b64 = input("ROM: ").strip()
encoded = input("output: ").strip().encode()

rom = base64.b64decode(rom_b64)
custom = rom[-64:]
canonical = (
    string.ascii_uppercase
    + string.ascii_lowercase
    + string.digits
    + "+/"
).encode()

body = encoded.rstrip(b"=")
translated = bytes(canonical[custom.index(ch)] for ch in body)
translated += b"=" * (-len(translated) % 4)

compressed = base64.b64decode(translated)
print(zlib.decompress(compressed).decode())
```

这里不能只根据可打印字符串猜“换表 Base64”：真正的验证是映射后的数据能通过 Base64 解码和 zlib 校验，并解压出结构正确的 flag。第一问输出中的提示文本 `N0w_l34rn_uxn!!!` 进一步指向第二、三问所用的 Uxn VM。

### 第三问：恢复位排列与循环密钥

第三个 ROM 的尾部字母表是不会执行到的调试残留，继续套用前两问会走错方向。其有效路径可归纳为：

1. `on-argument` 把运行参数写入零页，作为循环密钥；`on-stdin` 才处理 zlib 数据。
2. `core` 逐位取出输入，每累计 24 位形成一组。写入位置使用递推
   $p_{i+1}=(13p_i+7)\bmod 24$，因此三个输入字节的位被打散。
3. `zip` 把 24 位状态重新拼成四个 6 位值。
4. `putchar` 先加 `0x30`，得到 `0x30` 到 `0x6f`，再由 `hexify` 拆成高、低两个半字节。
5. `vig` 将每个半字节与循环密钥对应字节的低 4 位相加，模 16 后用 `a` 到 `p` 输出。

因此每 3 个压缩数据字节会变成 8 个 `a`～`p` 字符。未知密钥看似需要枚举 $16^L$ 种可能，但每个 6 位组加 `0x30` 后的高半字节只能是 3、4、5、6。对输出的偶数位置应用这个约束，就能把每个密钥位置的候选低半字节大幅缩小。题目生成的密钥长度为奇数，官方脚本需要检查 11 和 13 两种长度；候选解经逆排列后直接交给 zlib，校验和会排除错误密钥。

下面是整理后的完整求解脚本：

```python
import itertools
import zlib

cipher = [ord(ch) - ord("a") for ch in input("output: ").strip()]


def key_candidates(length):
    candidates = [list(range(16)) for _ in range(length)]
    for i, value in enumerate(cipher):
        key_pos = i % length
        if i % 2 == 0:  # 加 0x30 后的高半字节
            candidates[key_pos] = [
                k for k in candidates[key_pos]
                if (value - k) % 16 in range(3, 7)
            ]
    return candidates


def undo_bit_permutation(bits):
    original = ["0"] * 24
    source = 0
    target = 0
    while True:
        original[target] = bits[source]
        source = (source * 0x0D + 7) % 24
        target += 1
        if source == 0:
            break

    return bytes([
        int("".join(original[7::-1]), 2),
        int("".join(original[15:7:-1]), 2),
        int("".join(original[23:15:-1]), 2),
    ])


for key_length in (11, 13):
    pools = key_candidates(key_length)
    for key in itertools.product(*pools):
        plain_nibbles = [
            (value - key[i % key_length]) % 16
            for i, value in enumerate(cipher)
        ]

        bits = ""
        valid = True
        for i in range(0, len(plain_nibbles), 2):
            high, low = plain_nibbles[i:i + 2]
            if high not in range(3, 7):
                valid = False
                break
            bits += f"{high - 3:02b}{low:04b}"
        if not valid or len(bits) % 24:
            continue

        compressed = b"".join(
            undo_bit_permutation(bits[i:i + 24])
            for i in range(0, len(bits), 24)
        )
        try:
            result = zlib.decompress(compressed)
        except zlib.error:
            continue

        if result.startswith(b"flag{"):
            print(result.decode())
            raise SystemExit
```

脚本的三重验证条件分别是：半字节范围满足程序结构、重组结果通过 zlib 格式与校验和、解压内容具有 flag 格式。它们比仅凭可打印输出筛选密钥可靠得多。

## 方法总结

- 核心技巧：通过 Uxn Console 设备访问定位入口和 I/O，前两问提取 ROM 尾部的 64 字节表，第三问逆向位排列并用输出高半字节约束削减循环密钥空间。
- 识别信号：看到“第一行字节码、第二行运行结果”、`0x0100` 入口和对 `0x10`～`0x18` 设备页的访问时，应优先按 Uxn/Varvara VM 分析，而不是把所有可打印输出都当作多层编码。
- 复用要点：自定义编码的可靠终点应由下游格式校验确认；本题可以利用 Base64 结构、zlib 校验和以及 `flag{` 前缀形成逐层验证链。反汇编中的尾随数据不一定参与执行，必须先用控制流和引用关系确认表是否有效。
