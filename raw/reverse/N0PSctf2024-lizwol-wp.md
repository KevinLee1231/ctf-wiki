# Lizwol

## 题目简述

附件包含压缩器 `lizwol` 和仅 179 字节的 `flag.enc`。程序实现的是自定义 LZW，而不是密码算法：初始字典包含全部单字节，代码宽度由 8 位动态增长，字典最多有 $2^{20}$ 个条目。

仓库给出的简短官方说明只写了 `./solve flag.enc`，但实际解码器需要输入、输出两个路径。本篇依据编码器、官方 `solve.c` 和运行结果补全格式细节与可执行解码器。

## 解题过程

### 还原压缩格式

`lzw.h` 定义：

```c
#define DICT_SIZE (1 << 20)
#define CODE_NULL DICT_SIZE
```

编码状态初始为：

```c
ctx->code     = CODE_NULL;
ctx->max      = 255;
ctx->codesize = 8;
```

代码通过位缓冲区从高位到低位写出，所以是 MSB-first。每次加入新字典条目后，如果：

```c
ctx->max + 1 == (1 << ctx->codesize)
```

就把代码宽度增加 1。字典到达 `DICT_SIZE` 后直接重置，没有专门的 Clear 或 End 代码；文件结束时只把剩余位用 0 补齐到完整字节。

### 实现解码器

下面的 Python 代码复现官方 `solve.c` 的状态变化，包括“新代码等于 `max + 1`”时的 LZW 特例：

```python
from pathlib import Path


DICT_SIZE = 1 << 20
data = Path("flag.enc").read_bytes()
bit_position = 0


def read_bits(width: int) -> int | None:
    global bit_position
    if bit_position + width > len(data) * 8:
        return None

    value = 0
    for _ in range(width):
        byte = data[bit_position // 8]
        shift = 7 - bit_position % 8
        value = (value << 1) | ((byte >> shift) & 1)
        bit_position += 1
    return value


def reset():
    table = [bytes([value]) for value in range(256)]
    return table, 255, 8, None


table, maximum, code_width, previous = reset()
output = bytearray()

while True:
    code = read_bits(code_width)
    if code is None:
        break

    if code <= maximum:
        entry = table[code]
    elif code == maximum + 1 and previous is not None:
        entry = previous + previous[:1]
    else:
        raise ValueError(f"invalid code {code}")

    output.extend(entry)

    if previous is not None:
        table.append(previous + entry[:1])
        maximum += 1

    previous = entry

    if maximum + 1 == (1 << code_width):
        code_width += 1

    if maximum + 1 == DICT_SIZE:
        table, maximum, code_width, previous = reset()

Path("flag_recover.txt").write_bytes(output)
```

也可以直接使用仓库内的官方解码二进制，但必须补上输出参数：

```bash
./solve flag.enc flag_recover.txt
```

恢复文件包含逐步补全文本，最后一行为：

```text
NOPS{lzW_1S_a_c0mPr35sOr_f0R_5M41L_73xT_sTr!n9S!}
```

这里前缀确实是 `NOPS`，第二个字符是字母 `O`，而不是赛事常见的 `N0PS`。附件中的 `flag_recover.txt`、官方解码器输出和重新实现的 Python 解码结果三者一致，因此应保留证据中的原始拼写，不擅自修正。

## 方法总结

- 核心技巧：从 LZW 编码器恢复 MSB-first 位流、动态代码宽度和无控制码的字典重置规则。
- 识别信号：代码维护“前缀 + 字节”字典，初始索引为 0 到 255，并在找不到更长短语时输出当前代码。
- 复用要点：不同 LZW 变体的 Clear/End 代码、位宽增长时机和字典上限都可能不同，不能直接套用 GIF/PDF 解码器。恢复结果与赛事常见 flag 前缀冲突时，应优先相信多个独立本地证据。
