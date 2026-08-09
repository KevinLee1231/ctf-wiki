# CHAOS;HEAD

## 题目简述

程序实现带嵌套事务的命令解释器。`LOAD SNAPSHOT OPEN` 依次分配大小为 `0x800` 的 `SnapshotPage` 和 `0x780` 的 `SnapshotRoute`；后者保存导出函数指针 `sink` 和 `stdout`。大量嵌套 `BEGIN` 使事务容器扩容，`COMMIT` 后内部仍保留指向旧缓冲区的悬挂引用。若释放块被新的 `SnapshotPage` 复用，下一次 `BEGIN` 就会破坏正在使用的 page header。

该 UAF 同时提供两个原语：污染 `len` 后可越界泄漏最多 `0x1000` 字节；污染 `capacity` 后，`CLEAR` 不恢复容量，可以越界覆盖相邻 route 的函数指针。

## 解题过程

### 1. 触发事务悬挂引用

稳定触发序列为：

```text
CONFIG LOG ERROR
BEGIN × 55
COMMIT
LOAD SNAPSHOT OPEN
BEGIN
DUMP SNAPSHOT
```

前 55 次 `BEGIN` 迫使事务存储扩容，`COMMIT` 释放旧事务块却留下引用。`OPEN` 让 allocator 把该块复用为 `SnapshotPage`，最后一次 `BEGIN` 经悬挂引用写入其 header，使 `page->len` 和 `page->capacity` 变成异常大值。

### 2. 从 DUMP 泄漏地址

导出逻辑为：

```c
size = min(page->len, 0x1000);
route->sink(page->data, size, route->file);
```

`page->data` 从 page 偏移 `0x20` 开始，而 page 自身只有 `0x800` 字节。越界输出会覆盖到紧随其后分配的 `SnapshotRoute`，其中默认 `sink` 指向程序的 hex 输出函数，`file` 指向 libc 的 `stdout`。解析泄漏即可计算 PIE 与 libc 基址，同时保留 page 数据区地址作为后续伪造上下文的落点。

### 3. 覆盖 route 并切到 setcontext

执行 `LOAD SNAPSHOT CLEAR` 后，程序只重置 `len` 和 hash，没有恢复已污染的 `capacity`。于是可用 `LOAD SNAPSHOT DATA` 写入约 `0x900` 字节，越过 page 覆盖 route。相对 `page->data` 的关键位置为：

```python
payload[0x830:0x838] = p64(setcontext_0x3d)
payload[0x838:0x840] = p64(context_addr)
```

再次 `DUMP SNAPSHOT` 时，原本的 `route->sink(data,size,file)` 变成对 `setcontext+0x3d` 的调用。把伪造的寄存器上下文和 ROP 链布置在 page 数据区，使恢复后的 `rsp` 指向 ROP。

### 4. ORW 读取 flag

沙箱下不依赖交互 shell，ROP 链依次调用 `open("./flag", O_RDONLY)`、`read(fd, buf, size)`、`write(1, buf, size)`。文件描述符若不稳定，可用 `open` 返回值转移 gadget，或根据进程已打开描述符情况在本地确认后使用固定值。官方 `exp.py` 完成泄漏解析、route 覆盖、伪造 context 和 ORW。

## 方法总结

这道题的利用链建立在同一个 header 破坏之上，但读写原语需要分阶段使用：先利用大 `len` 泄漏布局，再 `CLEAR` 保留大 `capacity` 完成覆盖。分析时应把对象尺寸和字段偏移画清楚，确认泄漏中的 route、函数地址和 `stdout` 各自对应哪个基址；控制流劫持后优先采用 ORW，避免把题目环境是否提供 shell 作为额外不确定因素。
