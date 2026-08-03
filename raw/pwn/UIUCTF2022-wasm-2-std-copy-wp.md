# WASM 2: std::copy

## 题目简述

第二问给 WasmEdge 0.10.0 打了一个补丁：Universal WASM 的 AOT 段仍会被解析和装载，但在设置函数符号前强制执行 `IsUniversalWASM = false`，因此第一问直接替换宿主机器码的办法不再能触发。

补丁位置过晚。AOT 内容此前已经进入 `SharedLibrary::load()`，其中存在未校验的复制：

```cpp
BinarySize = 0;
for (const auto &Section : AOTSec.getSections()) {
    BinarySize = std::max(
        BinarySize,
        std::get<1>(Section) + std::get<2>(Section)
    );
}

Binary = Allocator::allocate_chunk(BinarySize);

for (const auto &Section : AOTSec.getSections()) {
    const auto Offset = std::get<1>(Section);
    const auto &Content = std::get<3>(Section);
    std::copy(Content.begin(), Content.end(), Binary + Offset);
}
```

AOT section 中的 `Offset`、声明尺寸和 `Content` 都来自攻击者输入，但复制时没有验证目标区间完整落在 `Binary` 分配范围内。于是，恶意 Universal WASM 即使不会执行 AOT 代码，仍能在加载阶段把受控字节写到 `Binary` 映射之外。

## 解题过程

### 从相对越界写到 glibc FILE 泄露

`Binary` 是 WasmEdge 分配的大块 mmap 内存。在题目固定的 Ubuntu 20.04 进程布局中，这类映射与共享库映射之间具有可重复的相对距离。因此，可先在本地使用随题提供的 `chal`、`libwasmedge_c.so` 和远端 Dockerfile 对齐环境，再用调试器记录：

```text
Binary 映射基址
libc 基址
_IO_2_1_stdout_ 地址
目标 FILE 字段相对 Binary 的偏移
```

第一阶段恶意 AOT 段把写入落到 stdout 的 `_IO_read_end`、`_IO_write_base` 等字段。题目官方解法引用的[同类 FILE 利用记录](https://github.com/bennofs/docs/blob/9a0ef37d6037836d66e23288a8aa461b516c0fee/hxp-2017/impossible.md)说明了具体原理：无缓冲 stdout 的指针通常指向 FILE 内部 `_shortbuf`；把写入起点向前调整到 `_markers` 一带，下一次 flush 会把 `_chain` 等 FILE 内部指针作为缓冲区内容输出，由此恢复 libc 地址。该外部案例并非本题现成 payload，但其关键的 mmap 相对布局和 FILE 泄露机制已经在这里说明。

获得 libc 基址后，可以进一步把 stdout 缓冲区指针指向 libc 数据段中的指针字段，以形成定向读；例如 `program_invocation_name` 一类全局量通常保存指向初始栈的地址。这样便能恢复栈地址，并结合本地 `chal` 反汇编确定 `main` 栈帧、保存返回地址以及当前循环所处位置。

### 利用同一连接完成第二阶段写栈

服务与第一问的另一个差异是外层存在循环：

```c
printf("Run another file? (y/n): ");
scanf("%c", &go_again);
if (go_again != 'y') break;
```

第一次加载恶意文件完成 stdout/libc/栈泄露后，回复 `y`，便可在同一进程、同一 ASLR 布局中提交第二个 AOT 文件。第二阶段按已知的栈地址重新计算 `Binary + Offset`，把 ROP chain 写到 `main` 的保存返回地址附近。ROP 可调用 libc 中的 `system("/bin/sh")`，也可设置寄存器调用 `execve`；最后回复 `n` 或安排函数返回，触发被覆盖的控制流，再读取 `/flag`。

完整利用应按以下顺序验收：

1. 用越界写修改 FILE 后，确实从后续 `printf` 获得 libc 指针；
2. 根据该指针计算的 libc 基址与随环境对应的 ELF 映射一致；
3. 再经 FILE 定向读获得栈指针，并核对它落在 `[stack]` 映射；
4. 第二个 AOT 文件写入保存返回地址和完整 ROP chain；
5. 退出循环触发 ROP，而不是在 `WasmEdge_VMDelete` 等更早路径中崩溃。

仓库提供了漏洞源码、补丁和上述官方利用路线，但没有提交最终 exploit；`healthcheck.py` 也明确保留 `TODO: add real healthcheck`。因此这里不能把某组未验证的 libc 偏移伪装成可直接运行的 solver。题目元数据记录的预期 flag 为：

```text
uiuctf{wh0_kn3w_l0ad1ng_4_b1n4ry_c0u1d_b3_s0_d4ng3r0us?_d86e520d}
```

漏洞位置可在题目锁定版本的[WasmEdge 源码](https://github.com/WasmEdge/WasmEdge/blob/312d7628a3dce72431abc39d88aa17745bdf612d/lib/loader/shared_library.cpp#L101)中交叉核对；正文已经概括了该链接最重要的信息，即 `Content` 被复制到攻击者控制的 `Binary + Offset` 而没有边界验证。

## 方法总结

- 核心技巧：利用 AOT loader 的越界 `std::copy` 获得相对大块 mmap 的受控写，先破坏 glibc stdout FILE 结构泄露 libc 与栈，再在同一进程的第二轮输入中写栈并执行 ROP。
- 识别信号：补丁只禁止使用已加载对象，却没有禁止解析不可信 AOT 段；分配尺寸与实际复制长度来自不同字段，且目标地址使用攻击者可控偏移。
- 复用要点：禁用危险功能必须发生在解析和资源分配之前。对序列化本机代码、cache 或 AOT 文件，应同时验证整数溢出、`offset + length`、声明长度与实际内容长度，并避免把攻击者偏移直接用于裸指针运算。
