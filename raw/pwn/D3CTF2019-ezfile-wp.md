# ezfile

## 题目简述

题目是 Ubuntu 18.04/libc 2.27 环境下的菜单堆题，功能包括 add、delete 和一个干扰性的 encrypt。程序禁用 `execve`，且除了第一次 `printf(name)` 外其余输出走 `write`，限制了常规 IO leak。漏洞是 tcache double free 加只能覆盖到返回地址的栈溢出；利用重点是改 `stdin` 的 `fileno`，让后续输入/输出落到程序开过的 `/dev/urandom` 文件描述符，再控制返回地址调用 `open("flag", 0)` 并把 flag 内容打印出来。

题目资料地址：https://github.com/hu4i/D3CTF_2019_ezfile

`ezfile.c` 启动时打开 `/dev/urandom`，读入 name 后 `printf("welcome!%s")`，再把随机数据读入堆上的 `key` 并关闭 fd；seccomp 同样禁止 `execve`。`deleteNote()` 释放 `notes[i]` 后只修改状态、不清指针，形成 double free；`encryptNode()` 中局部 `seed[0x50]` 在异常 size 下最多读 `0x70`，存在可控栈溢出。源码里这些点对应正文中的 tcache double free、半字节爆破和返回地址控制。

## 解题过程

- 标准菜单题，功能为add和delete。还有一个encrypt，但是并没卵用。
- 题目禁止使用execve， 最开始时会调用open打开/dev/urandm，之后scanf读入名字，然后printf名字，读取/dev/urandm然后close。
- 漏洞是一处tcache double free和栈溢出(只能覆盖到返回地址）。
- 保护措施： 只关了canary
- 除了第一次printf之外其他的输出函数都使用write（防止用IOfile泄露地址）

题目利用链：  
double free 修改stdin 的 fileno 为 3 （要猜半个字节，概率16分之一）  
栈溢出控制返回地址为open，其中rdi和rsi可以控制。 （也要猜半个字节，概率16分之一）。 从encrypt函数中让rdi == addrof(“flag”) rsi = 0 。然后scanf的stdout会从3中读取输入，printf打印出来。

## 方法总结

- 核心技巧：在禁用 `execve` 且常规输出受限时，通过 tcache double free 改 `_IO_FILE` 的 `fileno`，把已有文件描述符纳入利用链。
- 识别信号：程序启动时打开敏感 FD、后续使用 `scanf`/`printf` 或 FILE 流、同时存在 double free 时，应检查能否篡改 `stdin/stdout` 的 `fileno`。
- 复用要点：本题两处半字节爆破各约 1/16，整体 exploit 需要容忍概率失败；无法 `system("/bin/sh")` 时，优先走 `open/read/write` 或复用程序已有 IO 路径读 flag。

