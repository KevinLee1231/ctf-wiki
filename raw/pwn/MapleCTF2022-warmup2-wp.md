# warmup2

## 题目简述

程序连续询问姓名和年龄，两处都对 256 字节栈缓冲区执行超长读取，随后用 `%s` 回显。远端实际启用了 stack canary、PIE、NX 和 ASLR；要完成利用，需要依次泄露 canary、PIE 与 libc，并让程序多次回到输入流程。

## 解题过程

姓名缓冲区到 canary 的偏移为 `0x108`。canary 的首字节通常为 `0x00`，用于截断字符串；填满缓冲区并把该零字节覆盖为标记 `~`，`%s` 就会继续打印后七字节。接收后在前面补回 `\x00`：

```python
io.sendafter(b"What's your name?\n", flat({0x108: b"~"}))
io.recvuntil(b"~")
canary = u64((b"\x00" + io.recvuntil(b"!\n", drop=True))[:8])
```

年龄输入时放回 canary，并把返回地址低字节改到 `main` 中再次提问的位置。第二轮把字符串终止点推到保存的代码指针前，泄露 PIE 指针并减去已知偏移 `0x12e2`。有了程序基址后构造 ROP，调用 `puts(puts@got)` 再返回 `main`，由泄露值计算 libc 基址。

最后一轮保留正确 canary，使用 libc 中的 `ret` 对齐栈，再调用 `system("/bin/sh")`：

```python
rop = ROP(libc)
rop.raw(rop.ret)
rop.system(next(libc.search(b"/bin/sh\x00")))
```

读取 flag 得到：

```text
maple{we_have_so_much_in_common}
```

## 方法总结

`%s` 回显与不终止的超长读取组合后既能溢出，也能把栈保护值变成信息泄露。利用链应分阶段维护同一进程状态：先 canary，再 PIE，再 libc，最后 ROP；每轮覆盖都必须恢复 canary。部分返回地址覆盖在这里用于建立循环，而不是直接完成最终劫持。
