# unprintableV

## 题目简述

题目是格式化字符串 pwn，环境使用 Ubuntu 18.04 的 libc 2.27。难点是没有可用 stdout，常规格式化字符串泄露无法直接打印出来，因此需要先恢复输出能力或改写文件描述符，再泄露 libc 并构造 ORW 读取 flag。

题目资料地址：https://github.com/koocola/d3ctf-unprintableV

`source/b.c` 的编译参数包含 `-fno-stack-protector -fPIE -pie -z now -z noexecstack`，并通过 seccomp 禁止 `execve`。程序先打印 `&a` 作为栈地址 gift，然后 `close(1)` 关闭 stdout；后续最多执行 100 次 `vuln()`，每轮 `read(0, buf, 300)` 后直接 `printf(buf)`，直到全局 `buf` 以 `d^3CTF` 开头才退出循环。公开 `exp.py` 也按这个思路先恢复 stdout，再泄露程序和 libc 地址，最后走 ORW 读取 `flag`。

## 解题过程

方法一 先修改栈上的buf地址指向io\_stdout，利用$hhn修改io\_stdout指针为io\_stderr(爆破4bit 1/16)，然后输出流就被开启了  
方法二 修改io\_stdout的fileno为2  
利用格式化字符串漏洞leak libc地址  
然后栈迁移到bss段上用open read write来读flag

## 方法总结

- 核心技巧：stdout 不可用时，先通过格式化字符串改 `_IO_2_1_stdout_` 或其 `fileno`，把输出重定向到可见的 stderr，再进入常规 leak/ROP。
- 识别信号：存在格式化字符串漏洞但 stdout 被禁用或不可见时，应优先检查 `_IO_FILE` 结构和文件描述符能否被部分覆盖。
- 复用要点：部分覆盖通常需要爆破低半字节或低字节，概率要写进 exploit 预期；拿到 libc 后可把栈迁移到 bss，使用 `open/read/write` 规避 `execve` 限制读取 flag。

