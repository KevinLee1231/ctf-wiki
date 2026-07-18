# week4junkcode

## 题目简述

附件 `junkcode.exe` 是 32 位 PE，未使用 UPX。程序包含干扰静态分析的花指令，但有效逻辑很短：要求输入 28 个十六进制字符，每两个字符转换为一个字节，再与 14 字节常量逐项比较。

## 解题过程

在主函数中先定位长度检查，程序要求：

```text
strlen(input) == 28
```

`0x40110a` 附近存在用于扰乱反汇编的分支。顺着实际可达路径继续分析，可进入 `0x401054` 附近的转换函数。该函数每次读取两个字符，把十六进制字符映射为 $0\sim15$，随后计算：

$$
b_i=(\operatorname{hex}(s_{2i})\ll4)\oplus\operatorname{hex}(s_{2i+1})
$$

高、低半字节没有重叠，因此这里的异或与按位或效果相同，本质就是常规十六进制解码。转换出的 14 字节数据会与 `0x403000` 处的常量比较：

```text
11 45 14 23 33 de ad be ef 13 57 19 bd 2a
```

所以不需要暴力枚举，也不必修改程序；直接把目标字节写成十六进制字符串即可：

```python
target = bytes.fromhex(
    "11 45 14 23 33 de ad be ef 13 57 19 bd 2a"
)

candidate = target.hex()
assert len(candidate) == 28
assert bytes.fromhex(candidate) == target
print(candidate)
print(f"0xgame{{{candidate}}}")
```

输出为：

```text
1145142333deadbeef135719bd2a
0xgame{1145142333deadbeef135719bd2a}
```

## 方法总结

花指令只影响反汇编器对控制流的判断，不是题目的数据变换。确认真实可达路径后，抓住“28 个十六进制字符 → 14 个字节 → 固定数组比较”这一主线，就能直接反向构造输入。原文件名中的 `iunkcode` 也已按源码目录和附件名称校正为 `junkcode`。
