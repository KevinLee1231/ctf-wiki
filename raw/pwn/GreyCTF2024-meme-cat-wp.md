# meme-cat

## 题目简述

程序分别分配两段 meme，再分配第三段内存逐字节拼接。复制循环会递增 `a`、`b`、`c` 本身，随后 `puts(c)` 和三次 `free` 使用的都不再是原始分配地址，形成越界读取与可控的 invalid free。目标环境为 glibc 2.31，可利用伪造 chunk、unsorted bin 泄露和 tcache poisoning 覆盖 `__malloc_hook`。

## 解题过程

核心错误是：

```c
for (int i = 0; i < x; i++) *(c++) = *(a++);
for (int i = 0; i < y; i++) *(c++) = *(b++);
puts(c);
free(c);
free(b);
free(a);
```

输入长度决定三个失效指针最终落在何处，输入内容又能在这些位置附近布置伪造的 `size` 和链表字段。官方利用分为四步。

第一轮在堆上喷洒 `0x431` 大小字段，使某个 invalid free 被 glibc 接受为伪造的大 chunk，并进入 unsorted bin。随后用空 meme 触发重用；`puts(c)` 从拼接区末端向后读出 unsorted-bin 指针。以题目附带 libc 的偏移 `0x1ecbe0` 计算 libc 基址。

第二轮用两个带 `0x21` 伪头的小块建立 tcache 条目，再次借助空拼接从块尾泄露堆指针。取得两项地址后，继续伪造相邻的 `0x111`、`0x121` chunk，让后续分配落入可控区域，并在拼接写入时覆盖一个 0x20 tcache 条目的 `next`：

```python
chunk1 = p64(0x121) * 8 + b"/bin/sh\x00"
chunk1 += b"B" * (0xa0 - len(chunk1) - 8)
chunk2 = p64(0x21) + p64(libc.sym.__malloc_hook)
```

glibc 2.31 的该 tcache 路径尚未使用 safe-linking，故伪造的裸指针可直接把下一次同尺寸分配导向 `__malloc_hook`。向该位置写入 `system` 后，堆中已经保存了 `/bin/sh`。

最后再次输入 meme 长度。`read_meme()` 调用 `malloc(len + 1)`，而 malloc hook 的第一个参数正是申请大小；把 `len + 1` 设置为 `/bin/sh` 的堆地址，就等价于执行 `system("/bin/sh")`。读取 flag 得到：

```text
grey{f4k3_chunk_17_71l_y0u_m4k3_17}
```

## 方法总结

该题的关键不是一次普通堆溢出，而是把“递增后的指针”精确落到攻击者布置的伪 chunk 上。利用链依次解决 allocator 不崩溃、libc 泄露、堆泄露、tcache 任意分配和 hook 触发。复现时必须使用题目附带的 glibc 2.31；换到移除 malloc hooks 或启用不同 tcache 防护的版本，链条不会原样成立。
