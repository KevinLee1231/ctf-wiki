# GlacierCTF 2024 wasmwbw

## 题目简述

题目用 Wasmtime 运行选手提交的 WebAssembly 模块，并导入两个危险 host function：

```text
inspect(address)           -> 从 /proc/self/mem 任意读 8 字节
write_byte_where(addr, b)  -> 向 /proc/self/mem 任意写 1 字节
```

`write_byte_where` 在真正写入前会把自身入口的前 4 字节改成 `ret`，因此只能使用一次。预期解法先用任意读定位进程、libc、栈和 Wasmtime JIT 代码，再用唯一的一字节写修改 JIT 生成的 memory64 边界检查，把模块内一个普通 8 字节 store 升级为可重复的任意地址写，最后在宿主栈上布置 ROP。

## 解题过程

### 1. 利用 inspect 建立地址视图

宿主以 `Config.wasm_memory64(true)` 开启 64 位线性内存。官方 payload 从预期 PIE 区域附近按大步长探测可读页，再向低地址逐页回溯 ELF 基址；随后利用题目固定构建中的全局/GOT 偏移取得：

- Wasmtime 宿主 ELF 基址；
- 动态生成代码映射 `xgen`；
- 模块线性内存对应的 native base，即 `module_offset`；
- GOT 中 `__libc_start_main` 指针，从而计算 libc 基址；
- 全局保存的 `argv` 栈指针，用于定位当前调用链返回地址。

每个候选地址都先用 `inspect()` 验证是否可读，并检查函数序言或地址范围，避免只依赖硬编码 ASLR 值。ELF/GOT 偏移和 libc gadget 偏移仍然与赛题附件版本绑定。

### 2. 在 JIT 代码中找到可改造的 store

Wasm 模块定义：

```zig
noinline fn victim(addr: u64, val: u64) void {
    const write_addr: *u64 = @ptrFromInt(addr);
    _ = inspect(@intCast(addr));
    write_addr.* = val;
}
```

正常情况下，这个 store 只能写入模块线性内存。Cranelift 为 memory64 地址生成运行时越界检查；题目版本的 x86-64 代码中包含 `cmova`/`cmovae`，越界时把实际目标寄存器替换为会导致 trap 的安全值。

为稳定定位函数，payload 在附近放入立即数标记 `0x42424242`，扫描其后的机器码，寻找首个低 16 位为 `0x470f` 或 `0x430f` 的条件移动，并排除扫描逻辑自身的检查。这样得到该指令的 ModR/M 字节地址。

### 3. 用一次 WBW 取消有效边界钳制

调用：

```zig
write_byte_where(victim_addr + 2, 0xc2);
```

只改条件移动的第 3 个字节，使指令不再覆盖真正参与 store 的地址寄存器。这里没有尝试第二次调用 host WBW；它的入口已经被四个 `ret` 字节禁用。

补丁后，传给 `victim()` 的 64 位 index 可以按：

$$
\text{index}=\text{target native address}-\text{module\_offset}
$$

计算。JIT 最终仍执行 8 字节 store，但目标变成任意宿主地址，于是获得可重复的 arbitrary write。

### 4. 覆盖返回地址执行 system

官方版本中，从已泄漏的 `argv` 指针减去 `0x120` 可定位待返回的栈槽。利用随题 libc 的固定偏移写入：

```text
pop rdi ; ret
libc 中的 "/bin/sh"
ret                 # 栈对齐
system
```

对应代码连续调用四次 `victim()`，每次写 8 字节。`entry()` 返回 `1234` 后，宿主输出 `Module returned: 1234`，随后返回路径落入 ROP 并启动 shell；发送 `cat /flag.txt` 得到：

```text
gctf{WBW_W3_Wr1t3_Wh4t_W3_W4nt_Wh3r3_W3_W4nt}
```

## 方法总结

本题把“一次任意一字节写”扩展为“无限次任意 8 字节写”的关键，是修改会反复执行的 JIT 代码，而不是试图绕过自毁的 host function。memory64 让模块能自然表达完整 64 位偏移，任意读负责消除 ASLR，最后才进入传统 ROP。运行不可信 Wasm 时，宿主绝不能导出 `/proc/self/mem` 读写能力；JIT 代码和宿主秘密也必须处于最小权限隔离进程中。
