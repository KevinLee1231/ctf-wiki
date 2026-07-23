# Shellcodia 2

## 题目简述

服务执行用户提供的 x86-64 shellcode，并检查当前目录是否创建了 `strange.txt`，且文件内容是否恰好为 `awesome`。需要自行完成 `open`、`write` 等系统调用。

## 解题过程

可以在栈上构造以空字节结尾的文件名和内容，然后调用 Linux x86-64 系统调用：

```asm
xor eax, eax
push rax
mov rbx, 0x7478742e65676e61
push rbx
sub rsp, 3
mov byte [rsp], 0x73
mov byte [rsp + 1], 0x74
mov byte [rsp + 2], 0x72

mov rdi, rsp
mov rsi, 65
mov rdx, 0x1a4
mov rax, 2
syscall

mov rdi, rax
mov rbx, 0x00656d6f73657761
push rbx
mov rsi, rsp
mov rdx, 7
mov rax, 1
syscall

xor rdi, rdi
mov rax, 60
syscall
```

其中 `65` 是 `O_WRONLY | O_CREAT`，`0x1a4` 是八进制权限 `0644`。文件写入成功后得到：

```text
UMDCTF{uu_rr_ G3tt1nG_g00d_w1tH_Th1s_$h3llc0de_stUff}
```

flag 中 `rr_` 后确有一个空格。仓库 README 的哈希已过期，以上文本与公开程序内的实际检查一致。

## 方法总结

系统调用型 shellcode 应明确 ABI：`RAX` 放调用号，参数依次使用 `RDI`、`RSI`、`RDX`。还要注意小端序、字符串终止符以及 `open` 返回的文件描述符必须传给后续 `write`。
