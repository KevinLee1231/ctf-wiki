# BYUCTF 2022 - Chicken

## 题目简述

附件 `chicken.chn` 每行只重复若干个 `chicken` 单词。每行的单词数就是一条指令，程序实现了一个栈式 esolang，并以非顺序方式把字符写入 flag 缓冲区。

## 解题过程

先逐行计算 `len(line.split(" "))` 得到操作码。仓库 `solve.py` 恢复的指令集为：

```text
n >= 10 : 压栈 n - 10
2       : 弹出两数并相加
3       : 弹出两数，计算较早值减较晚值
4       : 弹出两数并相乘
9       : 栈顶整数转 ASCII 字符
7       : 弹出 index，再弹出 char，把字符写到 flag[index]
```

解释器初始使用一串 `!` 作为定长输出缓冲，操作码 7 会按索引覆盖，因此不能简单收集每次生成的字符。最小解释器核心为：

```python
elif op == 7:
    index = stack.pop()
    char = stack.pop()
    flag = flag[:index] + char + flag[index + 1:]
```

顺序执行全部行后恢复：

```text
byuctf{th3r3_4r3_3ven_w0rs3_es0l4ngs_but_1m_lazzy}
```

## 方法总结

重复单词的数量承担操作码或立即数。恢复自定义 VM 时，应先写清栈顺序和每个操作的副作用；本题最容易错的是减法次序，以及把 opcode 7 误解为直接输出而忽略目标索引。
