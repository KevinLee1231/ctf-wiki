# UMDCTF 2019 - GOsh

## 题目简述

附件是去符号的静态 Go 1.10 ELF。程序把输入按 8 个字符切块，对块数和每个 64 位块执行同一套可逆混合函数，再与常量表比较。

## 解题过程

虽然 ELF 已 strip，`.gopclntab` 仍能恢复 Go 函数名和源文件线索。跟进输入处理可知：程序去掉换行，将输入转为 rune 序列，再切成 8 字符块。核心 64 位函数为：

```python
MASK = (1 << 64) - 1
A = 0x9ac2ea715dd96605
B = 0x0fdee6da14294fd9

def mix(value: int) -> int:
    value = value * A & MASK
    value = ~value & MASK
    value ^= value >> 49
    value = value * B & MASK
    value = value - (value << 45) & MASK
    value ^= value >> 35
    return value
```

两个乘数以及 $1-2^{45}$ 都是奇数，因此在模 $2^{64}$ 下存在乘法逆元；右移异或也可从高位向低位迭代恢复。逆函数的核心写法如下：

```python
def undo_xor_right(value: int, shift: int) -> int:
    result = value
    while True:
        updated = value ^ (result >> shift)
        if updated == result:
            return result & MASK
        result = updated

def unmix(value: int) -> int:
    value = undo_xor_right(value, 35)
    value = value * pow((1 - (1 << 45)) & MASK, -1, 1 << 64) & MASK
    value = value * pow(B, -1, 1 << 64) & MASK
    value = undo_xor_right(value, 49)
    value = ~value & MASK
    return value * pow(A, -1, 1 << 64) & MASK
```

块数目标 `0x69614cc3dd6f7cd6` 逆回去是 `5`。程序内的五个目标值分别为：

```text
ea001ec75b88068c d5c35ed9070d6672 a6c5b1d23aaada6d
61b3832fbbcf14d5 3099575c3b0162d6
```

逐个逆运算并按大端转成 8 字节，得到：

```text
UMDCTF-{
95x41~Z9
2$90@22_
g0l@Ng_i
5_sk4Ry}
```

拼接后的 flag 是：

```text
UMDCTF-{95x41~Z92$90@22_g0l@Ng_i5_sk4Ry}
```

原二进制接受该输入，SHA-256 也与官方摘要一致。

## 方法总结

Go 二进制即便 strip，`.gopclntab` 仍是恢复函数边界和名称的重要线索。自定义哈希不一定单向：若只由可逆的奇数乘法、取反和 xor-shift 组成，就能在固定宽度整数环上逐步逆转。求出候选后还应重新执行正向函数和原程序双重验证。
