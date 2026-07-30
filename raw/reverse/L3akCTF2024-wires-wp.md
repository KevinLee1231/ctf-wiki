# L3akCTF 2024 Wires Writeup

## 题目简述

二进制中嵌入了 500 条以 `ff` 开头的异常十六进制字符串。每条字符串表示一幅 $500\times500$ 高度图的一列，并用游程编码压缩：

```text
ff<count>ff<value>
```

`count` 表示当前数值连续出现的行数，`value` 是要填入矩阵的高度。恢复全部 500 列后，非零区域会直接组成 flag 字样。

## 解题过程

先从二进制提取纯十六进制长字符串。例如：

```bash
strings wires | grep -E '^ff[0-9a-f]+$' > message.txt
```

仓库的 `solution/out` 已保存了同样的 500 行数据。最简单的一行是：

```text
ff01f4ff00
```

其中 `0x01f4 = 500`，表示这一整列的 500 个元素均为 0。更复杂的列会连续拼接多个 `(count, value)`。

这里不能无条件使用 `line.split("ff")`：字段宽度并不固定，而且字段末尾的 `f` 与分隔符开头的 `f` 可能形成连续三个 `f`，导致分隔位置错位。每列解压后的长度必须恰好等于 500，这一约束足以唯一确定仓库内全部字符串的分段。

下面的脚本用带记忆化的解析器恢复每列，并把所有非零高度渲染成黑色：

```python
from functools import lru_cache
from pathlib import Path

from PIL import Image

lines = Path("message.txt").read_text().splitlines()
assert len(lines) == 500

matrix = [[0] * 500 for _ in range(500)]


def decode_column(line):
    @lru_cache(None)
    def parse(position, total):
        if position == len(line):
            return () if total == 500 else None

        if total >= 500 or not line.startswith("ff", position):
            return None

        # 仓库数据中的 count 使用 2 或 4 个十六进制字符。
        for count_length in (2, 4):
            separator = position + 2 + count_length
            if line[separator:separator + 2] != "ff":
                continue

            count = int(line[position + 2:separator], 16)
            if count <= 0 or total + count > 500:
                continue

            # value 的宽度可变，枚举当前数据可能的 1 至 4 位。
            for value_length in range(1, 5):
                next_position = separator + 2 + value_length
                if next_position > len(line):
                    continue
                if (
                    next_position < len(line)
                    and not line.startswith("ff", next_position)
                ):
                    continue

                value = int(
                    line[separator + 2:next_position],
                    16,
                )
                tail = parse(next_position, total + count)

                if tail is not None:
                    return ((count, value),) + tail

        return None

    result = parse(0, 0)
    assert result is not None
    return result


for column, line in enumerate(lines):
    row = 0

    for count, value in decode_column(line):
        for y in range(row, row + count):
            matrix[y][column] = value
        row += count

    assert row == 500


image = Image.new("L", (500, 500), 255)
pixels = image.load()

for y in range(500):
    for x in range(500):
        if matrix[y][x] != 0:
            pixels[x, y] = 0

image.save("decompressed-flag.png")
```

非零像素集中在矩阵中央的一条窄带。裁去空白边缘并放大后，结果如下：

![解压后的中央像素带清晰显示 L3AK flag 文本](L3akCTF2024-wires-wp/decompressed-flag.png)

读取得到：

```text
L3AK{42_1s_th3_answer}
```

## 方法总结

- 大量结构相似的长字符串通常是数据表而非普通提示；先统计条目数、公共前缀和每条解码后的总长度。
- RLE 的核心不只是识别 `(次数, 数值)`，还要处理字段宽度和分隔符碰撞。这里用“每列恰好 500 项”消除歧义。
- 高度数值本身不是 ASCII 文本；只需区分零与非零即可在二维平面看到字形。
- 视觉结果承载了最终证据，因此保留语义化命名的解压图；提取脚本等纯代码内容则写入正文，不另存截图。
