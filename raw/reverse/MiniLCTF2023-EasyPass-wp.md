# MiniLCTF2023 - EasyPass

## 题目简述

附件提供 LLVM bitcode `main.bc` 和一个自定义 Pass。Pass 把校验逻辑改造成以 NAND 为基础的虚拟机，并使用 IR 中函数的名称作为 opcode 数据。每个函数名编码 flag 的一个字节，共 26 字节。

## 解题过程

决定性信息不是运行时输入，而是 `llvm::Value::getName()` 取得的符号名。可以直接从 bitcode 中提取这些名字，也可以先生成目标文件再读符号表：

```bash
llc main.bc -o main.s
clang main.s -o main
readelf -sW main
```

按 Pass 的生成逻辑，后 13 个字节先与 `0xff` 异或；前 13 个字节分别与恢复后的后半段逆序异或。仓库 `exp.py` 给出的密文字节为：

```python
enc = [
    100, 4, 101, 15, 44, 93, 57, 35, 35, 0, 22, 5, 29,
    143, 147, 154, 160, 179, 147, 169, 146, 160, 175, 203, 140, 202,
]

for i in range(13, 26):
    enc[i] ^= 0xff
for i in range(13):
    enc[i] ^= enc[25 - i]

print(bytes(enc))
```

输出是：

```text
QwQ_s0oOo_simple_LlVm_P4s5
```

按题目格式得到：

```text
miniLctf{QwQ_s0oOo_simple_LlVm_P4s5}
```

## 方法总结

LLVM Pass 题不能只盯着最终 native 指令；Pass 可能把 IR 元数据、符号名或类型信息当成隐藏输入。本题先确定 `getName()` 的数据来源，再按生成器逆运算即可，无需完整模拟 NAND VM。若 bitcode 保留符号，`llc + readelf` 是比动态调试更稳定的提取路线。
