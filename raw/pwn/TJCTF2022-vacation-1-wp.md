# TJCTF2022 vacation-1

## 题目简述

`vacation` 在栈上只分配 16 字节缓冲区，却调用 `fgets(buf, 64, stdin)`，可覆盖保存的返回地址。程序未开启 PIE，并额外提供调用 `system("/bin/sh")` 的 `shell_land` 函数，因此这是一个直接的 ret2win 题。

## 解题过程

通过反汇编或循环图案可确认，从 `buf` 到返回地址的距离是 `0x18` 字节。目标函数地址固定，官方脚本使用 `shell_land + 5` 落到函数内合适的调用位置：

```python
payload = b'A' * 0x18 + p64(exe.sym['shell_land'] + 5)
io.sendline(payload)
io.interactive()
```

`vacation` 返回时，覆盖后的地址使控制流进入 `shell_land`，进程执行 `/bin/sh`。在交互 shell 中读取 `flag.txt`，得到 `tjctf{wh4t_a_n1c3_plac3_ind33d!_7609d40aeba4844c}`。

## 方法总结

这类入门栈题的完整验证链是：确认危险读入长度、测出返回地址偏移、检查 PIE 与目标函数地址、最后处理函数入口和栈对齐。题目已经提供可直接起 shell 的代码，没必要引入 libc 泄漏或复杂 ROP；选择最短、最稳定的控制流转移即可。
