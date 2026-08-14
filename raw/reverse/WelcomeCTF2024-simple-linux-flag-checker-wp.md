# Simple Linux Flag Checker

## 题目简述

附件是 Linux ELF flag 检查器。源码/反编译结果没有保存一条连续的 flag 字符串，而是在栈数组中逐字节赋值后调用 `strncmp` 比较。目标是从这些常量赋值中重建完整字符串。

## 解题过程

在反编译器中查看 `main`，可见局部数组按下标初始化：

```c
flag[0] = 'g';
flag[1] = 'r';
flag[2] = 'e';
flag[3] = 'y';
flag[4] = '{';
/* ... */
flag[43] = '}';
flag[44] = 0;
```

按索引顺序连接每个字符即可。也可以写下反编译器中的赋值常量并转换为字符，但无需猜测任何算法，因为检查函数明确比较前 44 字节：

```c
if (!strncmp(input, flag, 44))
    puts("correct");
```

重建结果为：

```text
grey{stack_strings_are_not_very_stringsable}
```

## 方法总结

编译器可能把局部字符串拆成一系列栈写入，使普通 `strings` 无法看到完整文本。反编译后按地址或数组下标重组常量，是恢复 stack string 的标准方法。
