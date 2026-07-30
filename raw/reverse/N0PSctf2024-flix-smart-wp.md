# Flix Smart

## 题目简述

仓库没有官方 WP，只提供自定义编码器 `felix_smart`、实现源码和 `flag.enc`。`build_flag.sh` 说明出题流程：

```bash
convert -background cyan -fill orange -pointsize 72 \
  -size 1500x caption:`cat flag.txt` \
  flag.tiff

./felix_smart flag.tiff > flag.enc
```

因此目标是逆向 `qwertycode.c` 的压缩格式，从 `flag.enc` 恢复 TIFF，而不是寻找内存破坏漏洞。

## 解题过程

### 还原自定义 LZW 参数

源码中的关键常量为：

```c
#define QWERTY_CLEAR   256
#define QWERTY_END     257
#define QWERTY_START   258
#define QWERTY_MINBITS 9
#define QWERTY_MAXBITS 12
```

实现本质上是 LZW 变体：

- 字典代码 `0..255` 对应单字节；
- `256` 清空字典，`257` 结束流，新条目从 `258` 开始；
- 代码宽度从 9 位逐步增加到 12 位；
- `writecode` 把代码从最高位开始装入 32 位缓冲区，因此比特流是 MSB-first；
- `felix_smart.c` 调用 `init_qwerty_write(-1, ...)`，负数会在初始化时转为默认 `earlychange = 1`；
- 字典到达 12 位容量上限前会输出 Clear，并重新回到 9 位。

解码时还要处理标准 LZW 的 `KwKwK` 情况：当前代码正好等于下一个尚未加入的字典索引时，条目应为 `previous + previous[:1]`。

### 编写解码器

下面的脚本与编码器的 MSB-first 位序和 early-change 边界一致：

```python
from pathlib import Path


CLEAR = 256
END = 257
MIN_BITS = 9
MAX_BITS = 12

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


def fresh_table():
    return [bytes([value]) for value in range(256)] + [None, None]


table = fresh_table()
code_width = MIN_BITS
previous = None
output = bytearray()

while True:
    code = read_bits(code_width)
    if code is None:
        raise EOFError("compressed stream ended before END code")

    if code == CLEAR:
        table = fresh_table()
        code_width = MIN_BITS
        previous = None
        continue

    if code == END:
        break

    if code < len(table) and table[code] is not None:
        entry = table[code]
    elif code == len(table) and previous is not None:
        entry = previous + previous[:1]
    else:
        raise ValueError(f"invalid LZW code {code}")

    output.extend(entry)

    if previous is not None:
        table.append(previous + entry[:1])

        # earlychange=1；解码器比编码器少维护一个待加入条目。
        if (
            len(table) == (1 << code_width) - 1
            and code_width < MAX_BITS
        ):
            code_width += 1

    previous = entry

Path("flag.tiff").write_bytes(output)
```

### 验证恢复结果

输出长度为 1,200,294 字节，开头四字节为小端 TIFF 魔数：

```text
49 49 2a 00
```

仓库同时保留了构建阶段的参考 `flag.tiff`，可做强校验。解码文件与参考文件的 SHA-256 完全一致：

```text
f36d9c422aa6f1946d8248f4a9d354370a6163cfd9c9da7ea2b689283541ed34
```

TIFF 只显示一行 flag 文字，没有额外视觉机制，因此直接转写：

```text
N0PS{H3re_iZ_tH3_3nc0Der_youuuu_l0ve!!!}
```

## 方法总结

- 核心技巧：从编码源码恢复 LZW 控制代码、变长位宽、MSB-first 打包方式和 early-change 边界，再实现逆向解码。
- 识别信号：存在 Clear/End 特殊代码、单字节初始字典以及“前缀代码 + 下一字节”扩展表，这是典型 LZW 家族结构。
- 复用要点：自定义 LZW 最容易错在位序、位宽增长时机和 `KwKwK` 特例。文件魔数只能做初步检查；有参考构建产物时，应再比较长度与完整哈希。
