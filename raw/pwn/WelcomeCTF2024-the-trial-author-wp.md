# The Trial Author

## 题目简述

程序先把最多 5 字节书名直接作为 `printf` 格式串输出，随后把堆上书页用 `strcpy` 复制到 0x100 字节栈缓冲区。目标是先泄露 libc 基址，再通过跨页非终止字符串覆盖返回地址并跳入 one-gadget。

## 解题过程

书名存在格式化字符串漏洞：

```c
printf(book_name);
```

输入 `%2$p` 可泄露栈上的 libc 指针。根据本题配套 libc 中该指针相对基址的固定偏移，计算 `libc.address`，再加上 `one_gadget` 给出的候选偏移得到目标地址。

第二处漏洞来自书页的固定大小读取与 `strcpy`：

```c
read(0, pages[i], PAGE_SIZE);
strcpy(page_to_print, pages[chosen_page]);
```

`pages` 指向同一块连续堆内存。若第一页恰好填满且没有 `\0`，`strcpy` 会越过页边界继续读取第二页，直到遇到空字节；目标栈缓冲区只有 0x100 字节，所以可覆盖返回地址。官方构建的偏移为 `0x138`：

```python
payload = b"A" * 0x138 + p64(libc.address + one_gadget_offset)
```

由于小端地址通常含空字节，难以放入长 ROP 链；one-gadget 只需覆盖一个返回地址，正好适配 `strcpy` 的终止特性。取得 shell 后读取：

```text
grey{strcpy_my_0ne_g4dg3t}
```

## 方法总结

这是一条“格式串泄露 ASLR → 跨页 `strcpy` 溢出 → one-gadget”利用链。连续堆页并不意味着 C 字符串会在页边界停止；只要缺少 NUL，复制就会继续。one-gadget 仍需针对题目 libc 版本并验证寄存器/栈约束。
