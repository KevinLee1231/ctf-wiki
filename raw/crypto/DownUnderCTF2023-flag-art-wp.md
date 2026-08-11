# DownUnderCTF 2023 flag art Writeup

## 题目简述

附件是一幅由 `. = w - o ^ *` 等字符组成的 ASCII 图。生成脚本对消息的每个字节依次计算模 2、3、5、7 的余数，再用余数索引调色板，把四个字符嵌入模板中的四个 `X` 位置。目标是从每四个有效字符恢复一个原始字节。

## 解题过程

先删除画布中的空格和换行，得到连续的有效字符流。设调色板为：

```text
.=w-o^*
```

每组四个字符在调色板中的下标分别满足：

$$
x\equiv r_2\pmod 2,\quad
x\equiv r_3\pmod 3,\quad
x\equiv r_5\pmod 5,\quad
x\equiv r_7\pmod 7.
$$

因为 $2\times3\times5\times7=210$，对常见 ASCII 范围足以唯一确定字符。也可以预先对所有可打印字符建立查找表：

```python
from pathlib import Path
from string import printable

palette = ".=w-o^*"
art = Path("output.txt").read_text().replace("\n", "").replace(" ", "")

lookup = {}
for value in printable.encode():
    signature = "".join(palette[value % modulus] for modulus in (2, 3, 5, 7))
    lookup[signature] = chr(value)

groups = (art[index:index + 4] for index in range(0, len(art), 4))
print("".join(lookup[group] for group in groups))
```

解码结果包含提示文本以及最终 flag：

```text
DUCTF{r3c0nstruct10n_0f_fl4g_fr0m_fl4g_4r7_by_l00kup_t4bl3_0r_ch1n3s3_r3m41nd3r1ng?}
```

## 方法总结

ASCII 图只是数据布局，核心是四个两两互素模数产生的剩余系。查找表方案直观且不依赖数学库；中国剩余定理方案则直接恢复同余方程的解。无论采用哪种方法，都必须只抽取模板中承载信息的字符，避免把排版空白混入分组。
