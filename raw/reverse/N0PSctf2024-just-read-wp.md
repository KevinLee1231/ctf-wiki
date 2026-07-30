# Just Read

## 题目简述

附件 `main` 是未去符号的 x86-64 PIE ELF。程序从 `argv[1]` 读取候选 flag，逐字符比较并输出成功或失败信息。没有加密、混淆或自修改，关键是准确阅读反编译条件并按零基索引重组 23 个字符。

## 解题过程

先确认文件类型：

```bash
file main
```

结果表明它是动态链接、未 stripped 的 64 位 ELF。反编译 `main` 后可以看到：

```c
input = *(char **)(argv + 8);

if (strlen(input) == 0x17 &&
    input[0]  == 'N' &&
    input[1]  == '0' &&
    input[2]  == 'P' &&
    input[3]  == 'S' &&
    input[4]  == '{' &&
    input[5]  == 'c' &&
    input[6]  == 'H' &&
    input[7]  == '4' &&
    input[8]  == 'r' &&
    input[9]  == '_' &&
    input[10] == '1' &&
    input[11] == 's' &&
    input[12] == '_' &&
    input[13] == '8' &&
    input[14] == 'b' &&
    input[15] == 'i' &&
    input[16] == 't' &&
    input[17] == 's' &&
    input[18] == '_' &&
    input[19] == '1' &&
    input[20] == 'N' &&
    input[21] == 't' &&
    input[22] == '}') {
    puts("Well done, you can validate with this flag!");
}
```

`0x17` 即十进制 23。按索引从 0 到 22 连接字符：

```text
N0PS{cH4r_1s_8bits_1Nt}
```

直接运行验证：

```bash
./main 'N0PS{cH4r_1s_8bits_1Nt}'
```

输出：

```text
Well done, you can validate with this flag!
```

官方原文的汇总表把索引整体写成从 1 开始，并漏掉了部分字符；本篇以实际反编译条件的零基索引为准，最终 flag 与程序验证结果一致。

## 方法总结

- 核心技巧：阅读 `main` 中的定长逐字节比较，按真实索引顺序重建输入。
- 识别信号：二进制未去符号，控制流简单，成功分支由一串常量字符比较直接支配。
- 复用要点：反编译器常把每个字节先复制到临时变量，再在一个很长的条件中比较。重写为 `input[index] == value` 后再整理，并用原程序执行验证，可避免索引偏移和漏字符。
