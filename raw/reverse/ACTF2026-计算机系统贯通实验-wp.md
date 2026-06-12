# ？！计算机系统贯通实验！？

## 题目简述

题目附件表面是一份物理实验报告 xlsx，实际在 Excel 公式中实现了一个 RISC-V 单周期 CPU。解法需要把 xlsx 当作 zip 解包，从工作表公式中恢复指令解码、ROM/RAM、寄存器和 MMIO 语义，再逆向虚拟机里的四段 flag 校验。

## 解题过程

将 xlsx 解包后，在 `xl/workbook.xml` 中能看到：

```text
absPath="E:\Projects\excelCPU\"
```

继续分析 `xl/worksheets/sheet1.xml`，各列功能为：

```text
I 列 Row 2~3816      混淆后的 RISC-V 指令，共 3815 条
K 列 Row 2325~4509   ROM 常量
J 列                 寄存器文件
L 列                 RAM
B2                   reset 信号，设为 0 后按 F9 单步执行
```

地址映射为：

```text
address = (row - 2) * 4
```

K 列第一个非零数据在 Row 4098，对应地址 `0x4000`。指令解码公式来自 `C6` 等单元格：

```text
inst = HEX2DEC(I[row]) - 11462713 * (PC / 4) - 918823512
```

解码后得到标准 RISC-V 32 位指令。程序通过向 MMIO 地址 `0x10000000` 执行 `sb` 输出字符。

逆向主循环后，flag 被切成四段独立校验。

第一段 `flag[0:35]`：前 31 字节做：

$$
t=((c\oplus 0x37)+47)\bmod 256
$$

与 `mem[0x454c]` 的密文比较，末 4 字节直接比较。逆运算为：

$$
c=((t-47)\bmod 256)\oplus 0x37
$$

得到：

```text
ACTF{do-u-love-General-Physics-(H)?
```

第二段 `flag[35:63]`：使用 `0x4114` 处 64 字节字母表：

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+-
```

再按：

$$
shuf[i]=alpha[(37i+47)\bmod 64]
$$

生成自定义 base64 表，`=` 为 padding。逆向解码 `0x4044` 处密文得到：

```text
I-cann0t-love-it-anymore233-
```

第三段 `flag[63:82]`：在工作区：

```text
expand 32-byte k\0
```

上滚动更新，每步设置：

```text
key[i % 6] = i
key[12..13] = flag[i..i+1]
```

然后计算 FNV 风格 32 位 hash，与 `0x416c` 起的 19 个 dword 比较。每步只涉及两个未知字节，可对可打印字符两两爆破并把 key 状态向后推进，得到：

```text
15c077f9-631f-44d8-b
```

第四段 `flag[83:99]`：使用 7 轮简化 AES-CBC。S-box 在 `0x4570`，密文在 `0x41b8`，IV 取 `flag[:16]`，即：

```text
ACTF{do-u-love-
```

轮密钥规则为：

$$
rk[i]=((key[i]\ll((r\&3)+3))\&0xff)\oplus(1\ll(r\bmod 10))
$$

MixColumns 使用非标准 GF(2^8) 约简多项式，`xtime` 溢出时异或低字节 `0x1d`，对应完整多项式 `0x11d`，不是 AES 标准的 `0x1b/0x11b`。最后一轮不做 MixColumns，从第 6 轮倒推：

```text
AddRoundKey
-> InvMixColumns (r <= 5)
-> InvShiftRows
-> InvSubBytes
```

最后异或 `iv ^ key`，得到：

```text
826-af6c60f15e4f
```

四段拼接并补上 `}`：

```text
ACTF{do-u-love-General-Physics-(H)?I-cann0t-love-it-anymore233-15c077f9-631f-44d8-b826-af6c60f15e4f}
```

把该 flag 输入 Excel VM，会输出：

```text
OK fine, nice you to get the flag
```

## 方法总结

- 核心技巧：把 xlsx 解包为 XML，从公式中恢复 Excel CPU 的指令、内存和输出语义，再对 VM 内的四段校验分别求逆。
- 识别信号：表格文件中出现 `absPath`、大量公式列、ROM/RAM 样式数据和 reset 单元格时，应检查是否是 spreadsheet VM。
- 复用要点：先恢复地址映射和指令解码，再处理具体校验；四段 flag 校验彼此独立时，应拆段求逆而不是尝试整体爆破。
