# CakeCTF2021 nostrings

## 题目简述

程序没有以普通字符串形式保存 flag，而是用输入字符作为二维表的行号，再检查：

```c
table[flag[i]][i] == flag[i]
```

发布表格塞入了大量形似 flag 的诱饵字符串，直接运行 `strings` 会得到许多假结果。但真正校验关系仍把每个位置的正确字符编码在“行号等于单元格字节值”这一对角性质中。

## 解题过程

### 确定表格布局

二进制中的表可视为 `unsigned char table[][127]`。当前 flag 长度为 58，但每行按 127 字节对齐保存，剩余位置为 NUL。可在数据区找到第一条可打印行开头的 `FakeCTF`；它对应 ASCII 0x20，也就是第 32 行。

从该位置起按 127 字节切分 95 个可打印字符行，再在前面补上 32 个空行，就能恢复行号与字节值的一一对应。也可以直接通过反编译器定位 `table` 符号并读取完整数组。

### 按列寻找自描述行

对每一列 $i$，枚举行号 $y\in[0,126]$。若 `table[y][i] == y`，那么 $y$ 就是该位置的 flag 字节：

```python
with open("distfiles/chall", "rb") as f:
    blob = f.read()

start = blob.find(b"FakeCTF")
raw = blob[start:start + (127 - 0x20) * 127]
raw = b" " * (0x20 * 127) + raw
rows = [raw[i:i + 127] for i in range(0, len(raw), 127)]

answer = bytearray()
for i in range(58):
    hits = [y for y in range(127) if rows[y][i] == y]
    assert len(hits) == 1
    answer.append(hits[0])

print(answer.decode())
```

发布附件实测输出：

```text
CakeCTF{th3_b357_p14c3_70_hid3_4_f14g_i5_in_4_f14g_f0r357}
```

诱饵行的具体文本并不重要；它们只负责让 `strings` 结果变得嘈杂，无法改变校验器必须保留的二维索引不变量。

## 方法总结

- 大量可疑字符串出现时，不应凭格式挑一个提交，而应沿实际比较指令恢复数据结构。
- 二维查表校验常把秘密编码在行列关系中；把反编译表达式改写为数学条件往往比逐条读诱饵更快。
- `sizeof`、行跨度和数据区对齐必须从当前二进制确认，不能把可见字符串长度误当成真实行宽。
