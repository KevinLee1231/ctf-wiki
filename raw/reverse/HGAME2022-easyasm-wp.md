# easyasm

## 题目简述

附件是一个 16 位 DOS 可执行文件，字符串区中的 `hgame{Fill_in_your_flag}` 只是占位符。校验循环会交换每个输入字节的高、低四位，再与 `0x17` 异或，并和数据段中的 28 字节目标数组比较。

## 解题过程

在 8 位字节中，交换高低半字节可以写成：

```text
swap(x) = ((x << 4) | (x >> 4)) & 0xff
```

该操作执行两次会回到原值，所以逆运算只需先把密文字节异或 `0x17`，再交换一次高低半字节。目标数组为：

```python
target = [
    0x91, 0x61, 0x01, 0xC1, 0x41, 0xA0, 0x60, 0x41,
    0xD1, 0x21, 0x14, 0xC1, 0x41, 0xE2, 0x50, 0xE1,
    0xE2, 0x54, 0x20, 0xC1, 0xE2, 0x60, 0x14, 0x30,
    0xD1, 0x51, 0xC0, 0x17,
]

def swap_nibbles(value):
    return ((value << 4) | (value >> 4)) & 0xff

flag = bytes(swap_nibbles(value ^ 0x17) for value in target)
print(flag.decode())
```

运行结果为：

```text
hgame{welc0me_to_4sm_w0rld}
```

原 PDF 给出了逆运算，公开参赛记录补全了目标数组和输出，可参见 [YuGao 的 Week1 复现](https://sxyugao.top/p/d379320f)。这里已将决定性常量、逆变换和最终结果全部写入正文，不依赖外链才能复现。

## 方法总结

分析 16 位汇编时要注意 `AL`、`AH` 与 `AX` 的关系，并始终把运算限制在 8 位。半字节交换是自反操作，因而无需爆破；从比较目标反向执行“异或、交换”即可恢复输入。
