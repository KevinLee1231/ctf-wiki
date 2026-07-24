# UMDCTF 2018 - EzPz

## 题目简述

附件是一个去符号的 64 位 x86 ELF，体积约 6.7 MB。程序把大量简单运算展开成极长代码，静态逐条追踪成本很高；题目所说的 “goofed” 在于最终明文仍直接留在栈上。

## 解题过程

`file` 和 ELF 头表明程序是固定地址的 `EXEC`，不是 PIE，因此反汇编地址可以直接用于调试。沿主执行路径找到最终返回前的位置 `0xa5e45d`，在此下断点：

```gdb
set pagination off
break *0xa5e45d
run
x/80bx $rbp-0x50
x/s $rbp-0x50
```

断点命中时，`$rbp-0x50` 指向已经构造完成的输出缓冲区。按字符串查看即可得到：

```text
Your flag is: UMDCTF-{IH0p3_YoU_d1dnT_SpEND_too_MucH_TimE_ON_TH1S}
```

因此 flag 为：

```text
UMDCTF-{IH0p3_YoU_d1dnT_SpEND_too_MucH_TimE_ON_TH1S}
```

## 方法总结

面对大规模展开或自动生成的代码，应先判断最终结果是否会在寄存器、栈或输出函数参数中聚合。动态运行到“消费结果”的位置，再检查缓冲区，通常比静态还原数百万条等价操作更可靠。
