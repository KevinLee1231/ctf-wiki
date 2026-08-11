# Ternary Brained

## 题目简述

附件 `message.bin` 不是普通文本：编码器先把一个 Brainfuck 程序视为三进制数字，再把大整数以最短 big-endian 字节串写出。三进制每两个数字表示一条 Brainfuck 指令：`00 >`、`01 <`、`02 +`、`10 -`、`11 .`、`12 ,`、`20 [`、`21 ]`。主要障碍是恢复这层自定义表示与 Brainfuck 语义，归入 Reverse。

## 解题过程

### 从 bytes 到指令流

将整个文件按 big-endian 转为整数，反复除以 3 得到三进制数字。若数字总数为奇数，在前面补一个 `0`，然后两两分组查表。编码器只生成八种合法二位对，因此恢复后是完整的 Brainfuck 程序，不需要猜测额外字符集。

官方 solver 的核心等价于：

```python
n = int.from_bytes(open("message.bin", "rb").read(), "big")
ternary = to_base(n, 3)
if len(ternary) % 2:
    ternary = "0" + ternary
program = "".join(mapping[ternary[i:i+2]] for i in range(0, len(ternary), 2))
```

### 执行 Brainfuck

对恢复的 `><+-.,[]` 指令按常规 8 位 tape 语义解释；编码器的 `generate` 为每个原文字符选择相邻 cell 加减或新 cell 写入的较短方案，再附加 `.` 输出。最终程序打印一段宣传文本，flag 位于该文本末尾。

### 验证

题目配置与源 `flag_text.txt` 中的验证值为 `DUCTF{i_hope_you_liked_that_parallel_universe_deGhie0ca8AFaivu}`。本文未执行 Brainfuck 或 solver；三进制映射、补零规则和输出来源均由 Go encoder、官方 WP 与 solver 静态复核。

## 方法总结

- 核心技巧：把“文件字节串”首先视为一个大端整数，确认其进制展开后是否能形成固定宽度 opcode，而不是把每个字节独立解码。
- 识别信号：题目同时给 encoder 与大整数文件，且 opcode 只含有限符号时，常见链是 base conversion → VM/语言解释。
- 复用要点：最短整数编码会丢失数值前导零，故必须按 opcode 宽度修复奇数长度；解码得到 Brainfuck 后再执行，不能把三进制字符直接当最终明文。
