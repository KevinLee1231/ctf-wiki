# N1CTF 2020 Echoserver Writeup

## 题目简述

题目运行在 32 位大端 PowerPC 环境，输入会被直接作为格式化字符串使用。赛时 qemu 配置意外关闭 NX，但官方利用仍展示了更完整的预期链：利用格式化字符串任意写，改写 `__malloc_hook`，先调用动态链接器函数把栈设为可执行，再跳转到 PowerPC shellcode 读取 flag。

## 解题过程

### 处理 PowerPC 大端格式字符串

先通过 `%p` 或带位置参数的格式串确定输入在栈上的序号，并泄露程序、libc 与栈地址。PowerPC 32 位采用大端序，构造地址和 shellcode时使用：

```python
context.arch = 'powerpc'
context.endian = 'big'
```

写入目标可拆成 16 位或 8 位，利用 `%hn`/`%hhn` 控制累计输出长度。所有地址都要按大端打包，否则格式串会把字节顺序相反的地址当作指针。

### 两阶段劫持 malloc_hook

题目环境中没有可靠的 `/bin/sh` 或 `/bin/cat`，单纯把 hook 指向 `system` 不够。官方脚本先改写 `__stack_prot` 为 7，并让 `__malloc_hook` 指向一个能间接调用 `_dl_make_stack_executable` 的位置。触发一次 malloc 后，动态链接器根据修改后的保护标志把当前线程栈改为可执行。

接着再次用格式化字符串把 `__malloc_hook` 改为栈上 shellcode 的地址，第二次 malloc 直接进入 shellcode。两阶段顺序不能颠倒，否则第一跳就会因 NX 失败。

### PowerPC ORW shellcode

shellcode 把文件名 `flag` 写入栈，依次调用 PowerPC Linux 的 `open` 与 `sendfile`/`read`、`write` 系统调用，把内容发回当前 socket。概念流程为：

```text
fd = open("flag", O_RDONLY)
sendfile(socket_fd, fd, NULL, size)
```

PowerPC 系统调用号、参数寄存器和 `sc` 指令与 x86_64 不同，必须按题目 ABI 组装。仓库 exploit 中的寄存器初始化和大端机器码是最可靠的基准。

### 区分赛时配置错误

由于比赛 qemu 实际未启用 NX，参赛者也可以直接跳到栈上 shellcode。该捷径不需要 `_dl_make_stack_executable`，但不能解释官方 exploit 的第一阶段。WP 应同时记录两者，并把配置错误明确标成非预期解。

## 方法总结

跨架构 Pwn 首先要固定 ABI：字节序、指针宽度、调用约定和系统调用表都会影响 exploit。格式化字符串只是写原语，真正的利用设计在于选择控制点和安排两次触发。环境保护状态必须从运行实例验证，不能仅依据 ELF 标记或本地 qemu 默认值推断。
