# 摆烂

## 题目简述

附件把多种隐写方式串在一起：文件尾部附加加密压缩包，表层 PNG 实际是双帧 APNG，两帧可用于盲水印提取；压缩包内的四张图是二维码碎片，扫码后又得到带零宽 Unicode 字符的普通段落。需要依次完成文件雕取、盲水印、二维码拼接和零宽字符解码。

## 解题过程

先用 `binwalk` 或 `foremost` 检查附件。文件同时包含 PNG 与 ZIP 数据，说明压缩包被附加在图片结束标记之后：

```bash
binwalk -eM challenge
# 或
foremost -i challenge -o carved
```

得到的 PNG 是 APNG。用 APNG 分帧工具拆出两张几乎相同的帧，再把它们作为原图和含水印图交给双图盲水印解码器。差分水印中的文字为：

```text
4C*9wfg976
```

这就是附加 ZIP 的密码。解压后得到四张二维码碎片；根据三个定位角、白色静区和模块网格将它们旋转、拼成完整二维码。扫码结果表面上是一段普通中文，但文本中混入了大量不可见字符。

官方 WP 只写到“零宽字符提取”，没有保留扫码后的载体文本。[公开复现记录](https://www.cnblogs.com/starme/p/18468791)补充了该段原文；其中实际使用四种字符作为四进制数字：

| 字符 | 数字 |
| --- | --- |
| `U+200C` ZERO WIDTH NON-JOINER | `0` |
| `U+200D` ZERO WIDTH JOINER | `1` |
| `U+202C` POP DIRECTIONAL FORMATTING | `2` |
| `U+FEFF` ZERO WIDTH NO-BREAK SPACE | `3` |

每 8 个四进制数字组成一个 16 位 Unicode 码元。无需依赖在线工具，可把扫码文本保存为 UTF-8 文件并运行：

```python
from pathlib import Path

text = Path("qrcode-text.txt").read_text(encoding="utf-8")
alphabet = "\u200c\u200d\u202c\ufeff"

hidden = "".join(char for char in text if char in alphabet)
digits = "".join(str(alphabet.index(char)) for char in hidden)

if len(digits) % 8 != 0:
    raise ValueError("零宽字符数量不是 8 的倍数，文本可能在复制时受损")

flag = "".join(
    chr(int(digits[offset:offset + 8], 4))
    for offset in range(0, len(digits), 8)
)
print(flag)
```

解码得到：

```text
hgame{1_W4nT_T0_p1Ay_r0Tten}
```

## 方法总结

本题每一层都提供了下一层的输入：文件结构给出压缩包，APNG 双帧给出口令，二维码给出文本载体，四种零宽字符再按四进制编码 flag。处理这类多层隐写时，应始终保留原始字节和中间文件；尤其是零宽字符，复制到会自动清洗 Unicode 格式控制符的平台后可能永久丢失，因此应优先保存扫码器的原始 UTF-8 输出。
