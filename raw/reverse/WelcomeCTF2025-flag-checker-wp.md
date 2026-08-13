# Flag Checker

## 题目简述

附件是一个无法正常运行的 ELF flag checker。源码使用 `-ffreestanding -nostdlib -static -s -O1 -T minimal.ld` 一类选项和精简链接脚本构建，删除了正常运行所需的部分结构，但比较函数与常量仍能被反汇编、反编译。程序要求输入长度为 36，并通过 22 个小函数逐步检查各字符。

关键混淆是把条件选择写成按位表达式。例如：

```c
if (flag[18] == (-(flag[5] == key[5]) & key[6])) {
    return checkflag_6(flag);
}
```

当比较为真时，`-(1)` 在二进制补码中是全 1，与目标字符按位与后仍是该字符；比较为假时结果为 0。对正常可打印 flag 而言，这等价于同时要求 `flag[5] == key[5]` 和 `flag[18] == key[6]`。

## 解题过程

先从入口确认长度约束和 `grey{` 前缀，再沿 `checkflag_0` 到 `checkflag_21` 逐个翻译条件。几个代表性约束如下：

```text
flag[5]  = 'f'    -> flag[18] = 'L'
flag[28] = '3'    且 flag[28] + flag[31] = 'c' -> flag[31] = '0'
flag[8] = flag[12] = flag[16] = flag[22] = flag[25] = flag[29] = '_'
flag[14] = 'H'    且 flag[14] + flag[9] = '|' -> flag[9] = '4'
flag[27] = 'h'    -> flag[7] = 'r'
flag[11] = 'l'    -> flag[26] = 'T'
flag[30] = 'w'    -> flag[33] = 'l'
flag[17] = 'f'    -> flag[20] = '9'
```

其余算术条件直接逆运算。例如：

$$
((\text{flag}[34]+3)\oplus0x67)=\texttt{'T'}
$$

可得 `flag[34] = '0'`；而：

$$
((\text{flag}[19]-0x2e)\oplus0x18)=\texttt{'+'}
$$

可得 `flag[19] = 'a'`。把全部位置依索引填回后得到：

```text
index:  00 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16 17
char :   g  r  e  y  {  f  0  r  _  4  1  l  _  7  H  e  _  f

index:  18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35
char :   L  a  9  5  _  O  f  _  T  h  3  _  w  0  R  l  0  }
```

拼接并用仓库中可运行的 `flag_checker_clean` 或源码的 `checkflag` 正向验证，最终结果为：

```text
grey{f0r_41l_7He_fLa95_Of_Th3_w0Rl0}
```

## 方法总结

- 核心技巧：即使 ELF 因异常链接脚本无法运行，也可从保留的检查函数和数据常量逆推出输入；`-(condition) & value` 是常见的无分支条件选择。
- 识别信号：一串短检查函数、固定长度输入、字符索引关系以及用负布尔值构造的位掩码。
- 复用要点：先把反编译表达式还原成布尔语义，再做代数逆运算；最终必须把候选输入送回原 checker 或等价 forward checker 验证。
