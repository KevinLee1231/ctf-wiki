# Nullullullllu

## 题目简述

程序运行于 Ubuntu 24.04 / glibc 2.39，主动泄露 libc 基址，并允许用户任选一个地址写入单个空字节：

```c
printf("libc_base = %p\n", (&puts - 0x87bd0));
scanf("%llx", (unsigned long long *)&mem);
*mem = 0;
```

除这一次 null-byte 写外没有普通堆对象或栈溢出。程序大量调用 `getchar()`，且 `stdin` 被设为无缓冲模式。可把 null 写落在 `stdin->_IO_buf_base` 的最低字节，使下一次底层读取覆盖 `stdin` 自身，再升级为任意地址写。

## 解题过程

### 把一个空字节扩展为 FILE 自覆盖

无缓冲的 `stdin` 初始 `_IO_buf_base` 指向 FILE 对象内部的一字节缓冲区，`_IO_buf_end` 只比它大 1。附件对应的 glibc 构建中，`_IO_buf_base` 的值最低字节非零；把保存该指针的字段地址作为 null 写目标：

```python
stdin = libc_base + libc.symbols["_IO_2_1_stdin_"]
target = stdin + 0x38  # &_IO_buf_base，按题目 libc 结构确认
```

公开实例中该字段地址等价于 `libc_base + 0x203918`。写零后，指针从 `...e963` 一类地址向下对齐到 `...e900`，恰好落到 `stdin` 的 `_IO_write_base` 附近，而 `_IO_buf_end` 保持原值。

后续 `getchar()` 最终进入 `_IO_new_file_underflow()`：

```c
count = _IO_SYSREAD(
    fp,
    fp->_IO_buf_base,
    fp->_IO_buf_end - fp->_IO_buf_base
);
```

原本只读 1 字节，现在会从更低地址读到旧的 `_IO_buf_end`，覆盖 `stdin` 的多个字段，包括 `_IO_buf_base` 与 `_IO_buf_end` 本身。

### 设置第二阶段任意写区间

根据“输入第几个 8 字节会落到 FILE 哪个字段”构造第一阶段数据，把新的：

```text
stdin->_IO_buf_base = target_begin
stdin->_IO_buf_end  = target_end
```

设为 libc 可写区中准备放置伪 FILE、`_IO_list_all` 指针等内容的一段连续区间。下一次 `getchar()` 触发 underflow 时，标准输入便被直接读入 `[target_begin,target_end)`，形成受长度约束但足够使用的任意地址写。

第一阶段必须同时保持 `_IO_read_ptr/_IO_read_end` 等字段能继续走到 underflow，并处理菜单 `getchar()` 与吞换行循环消耗的字节。实战中应在相同 libc 下逐字段对照，而不是按固定填充长度猜测。

### House of Cat / FSOP

在第二阶段写入中完成两件事：

1. 在 libc 可写区放置伪造的 `_IO_FILE_plus` 与 wide-data 结构；
2. 把 `_IO_list_all` 改为指向伪 FILE。

伪对象使用合法的 `_IO_wfile_jumps` 附近 vtable，以通过新版 glibc 的 vtable 检查；设置 `_wide_data`、`_lock`、mode 和函数指针，使退出刷新链最终调用 `system("sh")`。字符串可直接嵌入伪 FILE 的 `_flags` 附近，例如 `";sh;"`。

最后选择菜单 3 调用 `exit(1)`。`exit` 进入 `_IO_flush_all`，遍历被替换的 `_IO_list_all`，触发 wide FILE 调度并执行命令。取得 shell 后读取当前目录的 `flag.txt`。

逐字段的 glibc 2.39 偏移和完整 House of Cat 布局见 [Nullullullllu 专项 Writeup](https://pwn2ooown.tech/ctf/writeup/2024/06/11/r3ctf2024-NULL)。本文已包含从最低字节覆盖、FILE 自覆盖、第二阶段任意写到退出 FSOP 的完整因果链；外链主要用于核对该附件 libc 的精确结构偏移。

## 方法总结

单字节写的价值取决于被改指针后续如何参与长度计算。本题选择 `_IO_buf_base`，是因为 `_IO_buf_end-_IO_buf_base` 会把向下对齐的微小指针变化放大成 FILE 自覆盖；自覆盖又能直接改写下一次读取区间。glibc 的符号地址、`_IO_FILE` 布局和 wide-vtable 偏移都与具体构建绑定，必须使用附件或容器中的 libc 重新计算，不能照抄公开脚本的常量。
