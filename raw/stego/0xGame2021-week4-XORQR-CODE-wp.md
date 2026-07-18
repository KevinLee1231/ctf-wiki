# week4XORQR_CODE

## 题目简述

附件文本只包含 `-` 和 `,`，题目名提示 XOR 与二维码。对每个字符使用单字节密钥 `0x1c` 异或后可得到由 `0`、`1` 组成的方阵数据，将其渲染为黑白二维码即可扫描出 flag。

## 解题过程

两个字符的 ASCII 值只相差最低位：`-` 为 `0x2d`，`,` 为 `0x2c`。与 `0x1c` 异或后恰好变成字符 `1` 和 `0`：

```text
0x2d ^ 0x1c = 0x31  # "1"
0x2c ^ 0x1c = 0x30  # "0"
```

密钥 `0x1d` 会得到相反的黑白关系，因此也能在反色后恢复同一二维码。下面的代码过滤附件中的换行，执行异或，将位串按平方边长重排，并添加二维码静区：

```python
from math import isqrt
from pathlib import Path
from PIL import Image, ImageOps

raw = Path("cipher.txt").read_bytes()
symbols = bytes(value for value in raw if value in (ord("-"), ord(",")))
bits = "".join(chr(value ^ 0x1c) for value in symbols)

assert set(bits) <= {"0", "1"}
side = isqrt(len(bits))
assert side * side == len(bits)

qr = Image.new("1", (side, side), 1)
for index, bit in enumerate(bits):
    if bit == "1":
        qr.putpixel((index % side, index // side), 0)

qr = ImageOps.expand(qr, border=4, fill=1)
qr.resize((qr.width * 10, qr.height * 10), Image.Resampling.NEAREST).save("qr.png")
```

扫描生成的 `qr.png`，得到：

```text
0xGame{cyberchef_and_python_yyds}
```

## 方法总结

本题不是对整份文件做复杂密钥爆破，而是利用两个输入字符与 ASCII `0`、`1` 的数值关系确定单字节 XOR。恢复位串后还要验证长度能否组成正方形，并在渲染时使用最近邻缩放和白色静区，避免二维码模块被插值破坏。
