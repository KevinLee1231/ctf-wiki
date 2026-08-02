# baby-heap

## 题目简述

程序依次分配 `a=malloc(0x78)`、`b=malloc(0x10)` 和 `reader=malloc(0x78)`，并把 flag 从 `reader[1]` 开始读入。随后允许攻击者控制一次单字节越界写：

```c
a[0x78] = attack_size;
```

在 glibc 堆布局中，`a` 的用户区后面紧接 `b` chunk 的 `size` 字段低字节，因此可以伪造 `b` 的大小。程序之后释放 `b`，按用户给定大小重新分配，并断言新 chunk 与 `reader` 重叠。

## 解题过程

`malloc(0x78)` 对齐后的 chunk 大小是 `0x80`，所以 `a[0x78]` 正好命中下一块 `b` 的 size 低字节。`b=malloc(0x10)` 原 size 为 `0x21`；写入 `0x71` 后，分配器把它视为大小 `0x70`、`PREV_INUSE=1` 的 chunk。

释放伪造后的 `b`，它会进入 `0x70` tcache。随后请求 `0x60` 字节时，实际 chunk 大小同样为 `0x70`，因此取回 `b` 的地址。真实的 `reader` 用户区位于 `b+0x20`，新 chunk 的可访问范围于是覆盖 `reader`：

```text
a chunk       b/fake 0x70 chunk
              +0x00 user pointer
              +0x20 reader user pointer
                    +0x01 flag starts here
```

程序最终打印 `chunk[0x21]`，恰好等于 `reader[1]` 指向的 flag 字符串。利用脚本只需提交两个十进制整数：

```python
io.recvline()
io.sendline(b"113")  # 0x71，覆盖 b->size 低字节
io.sendline(b"96")   # malloc(0x60) -> 0x70 size class
print(io.recvall().decode())
```

输出为：

```text
tjctf{bby-eap-lol171296386}
```

## 方法总结

- 单字节堆溢出足以修改相邻 chunk 的 size 低字节，把一个小块伪装成跨越下一块的较大 chunk。
- 用户请求大小与 chunk size 不相同；本题的 `0x60` 请求加上头部并对齐后才进入 `0x70` tcache bin。
- 地址泄漏和程序断言已经提供布局验证，利用链的核心是“伪造 size → free 进错误 bin → 重新分配造成重叠 → 从偏移 `0x21` 读取 flag”。
