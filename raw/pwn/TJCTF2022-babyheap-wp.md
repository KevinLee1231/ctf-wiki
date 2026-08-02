# TJCTF2022 babyheap

## 题目简述

程序提供 `malloc`、`free`、`view` 三种堆操作，每个槽保存一个 `Slot { size, buf }`。`do_free` 释放数据缓冲区和 `Slot` 本身后没有将 `slots[idx]` 清空，因此同一索引仍可查看或再次释放，形成 use-after-free 与 double-free。程序使用 musl 1.1，并通过 seccomp 只允许 `open`、`read`、`write` 等少量系统调用，最终利用必须以 ORW 读取 flag，而不能直接 `execve`。

## 解题过程

先利用被释放的 `Slot` 与后续小块分配发生重用。官方脚本把伪造的 `size=100` 写入重用区域，再 `view(1)` 越界输出堆内容，由泄漏值减去固定偏移 `0x95610` 得到 musl 基址：

```python
malloc(0, 1, b'a')
malloc(1, 1, b'a')
malloc(1, 1, b'a')
free(1)
free(0)
malloc(2, 64, p64(100))
view(1)
libc.address = u64(io.recv(8)) - 0x95610
```

随后分两轮伪造 musl 1.1 的空闲块元数据。第一轮把一个伪块送入回收链，并让分配结果落到 `__stdin_FILE - 0x20` 附近；第二轮扩大伪块大小，再次操纵链表，最终取得对 `__stdin_FILE` 的覆盖能力。官方脚本所用的关键目标为 `libc.sym['__stdin_FILE']`，而不是 glibc 常见的 hook。

覆盖的 FILE 数据中放入 `flag.txt`、栈迁移 gadget 以及 ORW ROP 链。链依次执行：

```text
open("flag.txt", 0, 0)
read(3, __stdin_FILE, 128)
write(1, __stdin_FILE, 128)
```

选择菜单项 4 退出后触发 stdio 清理路径，控制流进入布置好的 gadget 与 ROP 链，输出 `tjctf{musl_1.1_not_stronk_enough_:pensive:_18ae7497e1b5a236}`。

## 方法总结

决定性漏洞是释放后槽指针仍保持可达，泄漏与任意写都由这一生命周期错误发展而来。musl 题不能套用 glibc tcache 模板，应以具体版本的块头、bin 链和 FILE 布局为依据。由于 seccomp 禁止 `execve`，在构造控制流之前就应把最终目标锁定为 `open-read-write`，避免得到执行权后才发现系统调用不可用。
