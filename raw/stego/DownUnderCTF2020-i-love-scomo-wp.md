# DownUnderCTF 2020 - i love scomo

## 题目简述

题目只给出一张看似普通的 JPEG，并明确提示使用了 steganography 和“hidden space”。载体本身没有必须保留的视觉线索；真正的机制是两层隐写：先从 JPEG 的 `steghide` 容器中取出文本，再用每行末尾“有空格/无空格”的差异恢复二进制消息。

## 解题过程

### 第一层：破解 steghide 口令

先确认图片中存在 `steghide` 可识别的数据，然后使用 `rockyou.txt` 对口令做字典尝试。StegCracker 的输出表明口令是 `iloveyou`：

```bash
stegcracker ilovescomo.jpg /usr/share/wordlists/rockyou.txt
```

也可以在得到口令后直接提取：

```bash
steghide extract -sf ilovescomo.jpg -p iloveyou
```

提取出的文件表面上只是澳大利亚国歌。直接阅读没有 flag，但用能显示不可见字符的编辑器观察，会发现部分行在换行符前多出一个空格。题目描述中的 **hidden space** 正是在提示这一层。

### 第二层：解码行尾空格

按官方编码规则，每行末尾有空格记为 `1`，没有空格记为 `0`。连续八个 `0` 是消息终止填充；在终止点前把 bit 串按 8 bit 分组并转成字节即可。

下面的脚本保留了行尾空格，只移除换行符：

```python
from pathlib import Path


def decode_whitespace(path: str) -> bytes:
    bits = []
    zero_run = 0

    for raw in Path(path).read_bytes().splitlines():
        bit = "1" if raw.endswith(b" ") else "0"
        bits.append(bit)
        zero_run = 0 if bit == "1" else zero_run + 1

        if zero_run >= 8:
            del bits[-8:]
            break

    usable = len(bits) - len(bits) % 8
    return bytes(
        int("".join(bits[i:i + 8]), 2)
        for i in range(0, usable, 8)
    )


print(decode_whitespace("national_anthem.txt").decode())
```

运行后得到：

```text
DUCTF{i_R3lLi_l0000O0oo0v3_5c0m0}
```

这里不要使用会自动裁剪行尾空白的复制方式或格式化工具，否则第二层载荷会在提取前被破坏。

## 方法总结

- 核心技巧：串联 `steghide` 字典破解与基于行尾空格的二进制隐写。
- 识别信号：图片被明确称为 steganography 载体，同时题面刻意强调 space；提取出的正常文本在可见内容上没有异常时，应检查空格、制表符和换行等不可见差异。
- 复用要点：多层隐写中，第一层解出的“正常文件”往往只是新载体；读取文本时只能去掉换行符，不能调用会吞掉行尾空格的 `strip()`。
