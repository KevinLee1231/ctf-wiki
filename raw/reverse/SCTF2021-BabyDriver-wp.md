# BabyDriver

## 题目简述

题目由 Windows 用户态程序、`key.bin` 和内核驱动组成。用户输入必须是 20 个字符，字符集映射为 6 位值；程序再把 120 个输入位重排为 15 字节，交给驱动检查四个模乘方程。

控制流被两类异常隐藏：空地址调用触发访问异常，VEH 修改 `RIP/RSP` 跳过伪代码；除零异常则读取 `RAX`，调用 `funcs[RAX]`。因此反编译结果中看似无效的 `c = index / 0` 实际是 3600 个位搬运函数的分发器。

## 解题过程

### 还原用户态位变换

`CheckFormat` 把字符映射为 0 至 63：数字为 0 至 9，大写字母为 10 至 35，小写字母为 36 至 61，`{`/`}` 映射为 62，`|` 映射为 63。每个字符只有低 6 位有效，正好拆成三个 2 位片段；20 个字符共 60 个片段。

每个 `funcN` 都把某个输入 2 位片段搬到 15 字节输出中的某个 2 位位置，并从全局 `totalC=3100` 减去一个权重。例如：

```c
void func0() {
    char x = gbuf1[3] >> (2 * 2);
    x &= 3;
    x <<= (3 * 2);
    gbuf2[8] |= x;
    totalC -= 144;
}
```

`key.bin` 的每个置位会触发对应函数。仓库的 `wp.py` 先把 3600 个函数按“输入片段 × 输出片段”重排，再用动态规划和 DFS 选择 60 个函数，约束为：每个输入片段使用一次、每个输出片段使用一次、权重总和等于 3100。由此生成的 450 字节位图正是当前仓库中的 `key.bin`。实际检查得到置位数 60、输入位置数 60、输出位置数 60、权重和 3100，说明它定义了完整的 2 位排列。

### 反解驱动目标字节

用户态通过 PEB 的保留槽传递通信结构，再调用 `SetSystemTime` 触发驱动注册的 `\Callback\SetSystemTime` 回调。写入检查前，驱动的 `fake1` 会改动末尾四字节：

```c
p[15] = p[14];
p[14] = p[13];
p[13] = p[12];
p[12] = 0x10;
```

随后把 16 字节解释为四个小端 32 位整数，检查

$$
a_i x_i\equiv b_i\pmod p,
$$

其中 $p=53816244564283$。逐项求逆得到：

```python
p = 53816244564283
a = [649430213, 895805425, 751586893, 3859015203]
b = [49033969837712, 36224070408864, 1911652611622, 32147829792607]

x = [(pow(ai, -1, p) * bi) % p for ai, bi in zip(a, b)]
assert x == [0xd4933333, 0x7bde8f00, 0x31d84f77, 0xbcd47c10]
```

第四个整数的低字节 `0x10` 是 `fake1` 插入的。按小端序撤销该插入与右移，用户态 `Trans` 应生成的 15 字节为：

```text
33 33 93 d4 00 8f de 7b 77 4f d8 31 7c d4 bc
```

### 逆 2 位排列

读取 `key.bin` 的每个 LSB-first 位，解析对应 `funcN` 的源字节/源 2 位位置和目标字节/目标 2 位位置，再把目标值放回输入位置。精简的恢复逻辑如下：

```python
import re
from pathlib import Path

key = Path("key.bin").read_bytes()
source = Path("client/func.cpp").read_text(encoding="utf-8")
target = bytes.fromhex("33 33 93 d4 00 8f de 7b 77 4f d8 31 7c d4 bc")

selected = [
    byte_index * 8 + bit_index
    for byte_index, value in enumerate(key)
    for bit_index in range(8)
    if value & (1 << bit_index)
]
assert len(selected) == 60

chunks = [None] * 60
weight_sum = 0
for index in selected:
    body = re.search(
        rf"void func{index}\(\)\s*\{{(.*?)\}}",
        source,
        re.S,
    ).group(1)
    src_byte, src_part = map(int, re.search(
        r"gbuf1\[(\d+)\]\s*>>\s*\((\d+)\s*\*\s*2\)", body
    ).groups())
    dst_part, dst_byte = map(int, re.search(
        r"x\s*=\s*x\s*<<\s*\((\d+)\s*\*\s*2\).*?gbuf2\[(\d+)\]",
        body,
        re.S,
    ).groups())
    weight_sum += int(re.search(r"totalC\s*-=\s*(\d+)", body).group(1))
    chunks[src_byte * 3 + src_part] = (target[dst_byte] >> (2 * dst_part)) & 3

assert weight_sum == 3100 and all(value is not None for value in chunks)
values = [
    chunks[3*i] | (chunks[3*i + 1] << 2) | (chunks[3*i + 2] << 4)
    for i in range(20)
]
```

反向字符表得到 `SCTF{Dr1ver|Tim3|10?`。值 62 同时代表 `{` 和 `}`，结合位置和标准格式把最后一位解释为右花括号：

```text
SCTF{Dr1ver|Tim3|10}
```

将该字符串重新执行字符编码、60 个选中函数、`fake1` 字节调整后，四个整数分别满足驱动中的模方程，程序返回 `Success!!!`。

## 方法总结

本题需要把异常驱动的用户态控制流、位排列约束和内核回调串在一起。除零异常并非崩溃，而是用 `RAX` 选择位搬运函数；`key.bin` 不是加密密钥，而是从 3600 个映射中选出一个总权重为 3100 的 60 元排列；驱动又在模方程检查前插入一个字节。最稳妥的逆序是先解驱动方程并撤销字节调整，再按选中函数反转 2 位排列，最后逆字符表，并用整条正向链验证。
