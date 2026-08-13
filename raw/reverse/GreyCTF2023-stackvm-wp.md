# GreyCTF 2023 StackVM

## 题目简述

题目要求在自定义栈虚拟机中实现斐波那契函数，通过从 0 到 560 的 35 组测试。指令只有 `NAND`、条件跳转、`SWAP`、`DUP`、`SHIFT`、`PUSH` 和 `POP`，程序最多 1000 行、总执行不超过 150000 步。解释器的 Python 下标语义还使负数 `SWAP` 参数能够访问栈底。

## 解题过程

`SWAP to fro` 实际访问 `stack[-1-to]` 和 `stack[-1-fro]`。当 `fro=-1` 时，第二个下标为 0，所以：

```text
SWAP 0 -1
```

可以交换栈顶与栈底，用来把初始参数移入循环状态。算术部分由 NAND 合成：`NAND(x,x)=~x`，再通过反相得到 AND；加法器使用

$x\oplus y=(x\operatorname{NAND}(x\operatorname{NAND}y))\operatorname{NAND}(y\operatorname{NAND}(x\operatorname{NAND}y))$

计算无进位和，以 `(x & y) << 1` 生成进位并循环，直到进位为零。外层循环维护相邻两个斐波那契数并递减计数器。

一份能通过全部测试的程序如下，跳转目标使用编译后的 0 起始指令行号：

```text
PUSH 1
PUSH 0
SWAP 0 -1
DUP
JE 56
SWAP 0 -1
DUP
DUP
SWAP 0 3
DUP
SWAP 1 2
NAND
DUP
DUP
SWAP 0 4
NAND
SWAP 0 2
NAND
NAND
SWAP 0 1
DUP
NAND
PUSH 1
SHIFT
SWAP 0 1
DUP
SWAP 0 2
DUP
JNE 9
POP
POP
SWAP 0 -1
PUSH 1
DUP
SWAP 0 2
DUP
SWAP 1 2
NAND
DUP
SWAP 0 2
NAND
SWAP 0 2
NAND
DUP
DUP
NAND
SWAP 0 2
NAND
SWAP 0 1
PUSH 1
SHIFT
DUP
JNE 33
POP
PUSH 0
JE 3
POP
POP
```

提交后 35 组输入的栈顶结果均与服务端 `fib` 一致，返回：

```text
grey{I_4m_st4ck3d_8d2eb04fa1d274e509e4ec9b70b240e65a0c61b0c1daaa81a1aace02e0b9bfa4}
```

## 方法总结

限制指令集并不意味着无法做通用算术：NAND 可构造任意布尔运算，SHIFT 提供进位移动，条件跳转组成循环。逆向这类 VM 时，应先精确记录每条指令的栈效果和跳转计数方式；同时检查宿主语言边界，本题的 Python 负下标显著简化了栈布局操作。
