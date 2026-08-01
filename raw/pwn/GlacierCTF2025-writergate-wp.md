# GlacierCTF 2025 writergate

## 题目简述

正式版提供三个 Zig 命令：`hash` 计算任意非 flag 文件的 SHA-256，`splat` 输出重复字符，`crash` 向用户给出的地址写一个字节后退出。`crash` 已经是显式的任意 byte-write，但 ASLR 下仍需泄漏 libc 和栈，并找到将后续输入写到返回地址的方法。

题目的核心仍是 Zig I/O 对象与底层缓冲区的别名：`hash` 的 Discarding writer 和 stdout 共用 `write_buffer`，可以泄漏 `/proc/self/maps`；随后利用栈地址和一字节写修改 `stdin.buf.ptr`，让 `crash` 最后的 refill 把 ROP chain 直接读到栈上。

## 解题过程

### 1. 组合 `splat` 与 `hash` 泄漏 maps

`write_buffer` 大小为 `0x600`。`splat` 将长度按十六进制解析，命令：

```text
splat 600   hash /proc/self/maps
```

包含三个连续空格。解析过程如下：

1. `600` 被解释为 $0x600$，stdout 的整个缓冲区被空格填满；
2. 第一个额外空格作为 splat 字节；
3. 第二个额外空格被代码误当作命令结尾消费；
4. 同一 stdin buffer 中剩余的 `hash /proc/self/maps` 在下一轮执行。

`hash` 使用：

```zig
var discarding_writer = Io.Writer.Discarding.init(&write_buffer);
```

它把文件内容写进与 stdout 相同的物理数组，却不更新 stdout 的逻辑状态。stdout 仍认为自己缓存着 $0x600$ 个待输出空格；后续打印 hash 结果触发 flush 时，实际数组已被 `/proc/self/maps` 覆盖，于是 maps 原文泄漏。解析其中 `libc.so.6` 的 mapping 起始地址和文件 offset，可计算 libc base。

### 2. 取得栈基准并定位 reader 指针

提交一个明显超过缓冲区长度的 splat：

```text
splat 123123123 A
```

错误分支会打印 `&write_buffer`，得到可靠的栈地址。参考 solver 针对附件二进制的相对布局为：

```python
stdin_buf_ptr_field = write_buffer - 0x23a8
saved_rip_area      = write_buffer + 0x650
```

这些偏移来自同一版本二进制的调试验证；若重新编译，必须重新测量，不能把它们当作 Zig ABI 常量。

### 3. 把一字节写升级成栈上长写

`crash` 的逻辑依次是：解析地址、读一个 payload 字节、执行 `v.* = b`，最后再调用一次 `stdin.takeByte()` 读取所谓换行，然后返回。

选择目标为 `stdin.buf.ptr + 1`，只修改缓冲区指针的第二低字节。由于原 read buffer 与保存返回地址都在同一栈区域，其余高位相同，一个字节足以把 refill 目的地移动到 saved RIP 附近。发送 `crash <addr> <byte>` 时故意不附加换行，确保执行任意写后 stdin 已无缓存，最后的 `takeByte()` 必须从 socket refill。

此时立即发送一个不超过 `0x100` 字节的 payload。refill 会把它写入被重定向的栈地址。为容忍指针落点和返回地址间的小偏差，payload 前部放置若干 libc `ret` 作为滑道，末尾使用已知 libc base 构造：

```text
execve("/bin/sh", 0, 0)
```

题目未启用 stack canary，函数返回时进入 ROP，获得 shell 后读取 `/flag.txt`：

```text
gctf{FSOPOnTheStackInYetAnotherWriteByteWhereChallenge}
```

## 方法总结

利用链是“write-buffer 别名泄漏 maps → splat 错误信息泄漏栈 → crash 任意一字节写 → 篡改 stdin buffer 指针 → 强制 refill 写栈 → libc ROP”。这里的 `hash` 过滤只禁止路径包含 `flag`，却允许 `/proc/self/maps`；更根本的问题是把同一数组交给两个拥有独立逻辑状态的 writer。安全修复既要移除任意地址写，也要为 hashing 使用独立 scratch buffer，并避免在错误信息中打印栈地址。
