# N1CTF 2020 EasyWrite Writeup

## 题目简述

程序先泄露 `setbuf` 地址，再分配一块约 `0x310` 字节的消息缓冲区。用户不仅能填写消息，还能指定一个地址，程序随后执行 `*addr = message_pointer`。最后程序分配并释放一块 `0x40` 大小的缓冲区。题目提供的是“把一个已知堆指针写到任意可写地址”的受限任意写，而不是直接写任意 8 字节。

## 解题过程

### 由函数泄露计算 libc

启动时给出的地址属于附带的 glibc：

```python
libc.address = leaked_setbuf - libc.sym['setbuf']
```

开启 Full RELRO 后 GOT 不可写；glibc 2.31 仍保留 `__free_hook`，因此最终控制点选它。

### 伪造 tcache_perthread_struct

程序的写原语只能写入第一块消息的地址。利用方法是先在这块大消息中伪造一个 `tcache_perthread_struct`：在与 `0x40` 请求对应的 bin 中，把计数设为 1，把链表头设为 `__free_hook - 8`。概念布局如下：

```text
counts[0x40-bin] = 1
entries[0x40-bin] = __free_hook - 8
```

glibc 的线程缓存指针位于 libc 可计算位置。把程序提供的任意写目标设为该位置：

```python
where = libc.address + TCACHE_POINTER_OFFSET
```

执行写入后，malloc 会把大消息中的伪造结构当作当前线程的 tcache 管理区。`TCACHE_POINTER_OFFSET` 必须从题目 libc、调试符号或运行时结构关系确定；公开脚本使用的是附件版本对应的固定偏移。

### 让最后一次分配覆盖 free_hook

程序随后申请 `0x40` 字节。由于伪造 bin 的计数非零，malloc 从 `entries` 取出 `__free_hook - 8`。给这块“缓冲区”写入：

```python
payload = b'/bin/sh\x00' + p64(libc.sym['system'])
```

前 8 字节落在 hook 前方，后 8 字节覆盖 `__free_hook`。程序马上执行 `free(last_message)`，于是调用等价于：

```c
system("/bin/sh");
```

公开的 [EasyWrite 详细分析](https://152334h.github.io/blog/n1ctf-2020-easywrite/) 给出了附件 libc 的具体偏移；上文已把偏移背后的结构关系完整展开。

## 方法总结

受限任意写的价值取决于“能写什么值”和“目标稍后如何被解释”。本题不能直接写 `system`，却能改写 tcache 管理结构的指针，使下一次 malloc 代替攻击者完成任意地址分配。面对新版本 glibc，应先确认 hook 是否仍存在以及 tcache 结构是否变化，再选择等价控制点。
