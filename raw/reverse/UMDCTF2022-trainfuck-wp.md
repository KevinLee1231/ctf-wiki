# UMDCTF2022 Trainfuck Writeup

## 题目简述

附件 `trainfuck.tf` 只由火车 emoji 组成。源码目录给出了 emoji 到 Brainfuck 指令的映射，以及生成该程序的伪 C 代码。程序逐字节检查输入：偶数位置与 42 异或，奇数位置与 69 异或，再和固定数组比较。

## 解题过程

先把八种火车字符还原为 Brainfuck：

```python
mapping = {
    "🚂": "+",
    "🚃": "-",
    "🚄": "<",
    "🚅": ">",
    "🚆": "[",
    "🚇": "]",
    "🚈": ".",
    "🚉": ",",
}

with open("trainfuck.tf", encoding="utf-8") as src:
    decoded = "".join(mapping[c] for c in src.read() if c in mapping)
```

直接解释完整 Brainfuck 可以验证输入，但恢复 flag 不必分析其冗长控制流。生成源码已经给出目标数组：

```text
127, 8, 110, 6, 126, 3, 81, 35, 88, 116,
73, 46, 117, 38, 66, 117, 26, 26, 73, 45,
26, 117, 117, 49, 93, 113, 27, 43, 89, 56, 10
```

前 30 项对应 flag 字符，末尾的 10 是换行。XOR 是自身的逆运算，逐位置解密即可：

```python
enc = [
    127, 8, 110, 6, 126, 3, 81, 35, 88, 116,
    73, 46, 117, 38, 66, 117, 26, 26, 73, 45,
    26, 117, 117, 49, 93, 113, 27, 43, 89, 56,
]

flag = "".join(
    chr(value ^ (42 if index % 2 == 0 else 69))
    for index, value in enumerate(enc)
)
print(flag)
```

输出：

```text
UMDCTF{fr1ck_ch00_ch00_tw41ns}
```

## 方法总结

本题有两层表示：emoji 只是 Brainfuck 的替换字母表，Brainfuck 又是由更清晰的校验逻辑编译而来。恢复映射能解释附件格式，但真正求解时应选择信息损失最小的表示层；已有生成源码时，直接逆向固定 XOR 比逐指令模拟 Brainfuck 更准确也更简洁。
