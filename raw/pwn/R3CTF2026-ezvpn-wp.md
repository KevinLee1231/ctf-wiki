# ezvpn

## 题目简述

题目是一个自定义 TLS VPN 服务。连接建立后，服务先发送：

```text
CLIENTINFO:
```

客户端回复 4 字节大端总长度，随后是若干 TLV 记录：

```text
[key_len:1][value_len:1][key bytes][value bytes]
```

之后才进入 `AUTH:` 用户名、密码认证。二进制中存在 `FOR_AGENT=R3CTF2026!` 特殊分支，它会返回一个看似正确的 flag，并泄漏 libc 中的 `_IO_2_1_stderr_` 地址。该 flag 是诱饵，地址泄漏才是有效信息。

真正漏洞位于 TLV 分配长度：程序用较宽整数检查 `key_len + value_len <= 0x100`，却把和截成 `uint8_t` 后再传给 `malloc`。令两者之和恰好为 `0x100`，检查通过而分配长度变为 0，随后发生可控堆溢出。

题目使用现代 glibc、PIE、Full RELRO、Canary、NX、IBT 和 Shadow Stack。预期解法通过堆溢出污染 tcache，把 libc 全局 `stdout` 指针改成指向攻击者缓冲区中的伪造 `FILE`，再以 `_IO_wfile_jumps` 触发 `setcontext`，最终在原进程中执行 ORW 读取 flag。

## 解题过程

### TLS 客户端信息格式

一条普通记录可以按以下方式编码：

```python
import struct

def record(key, value):
    if len(key) > 0xff or len(value) > 0xff:
        raise ValueError("field too long")
    return bytes([len(key), len(value)]) + key + value

def send_client_info(tls_socket, records):
    body = b"".join(records)
    tls_socket.sendall(struct.pack(">I", len(body)) + body)
```

特殊记录：

```python
record(b"FOR_AGENT", b"R3CTF2026!")
```

会进入代理分支。响应中的诱饵 flag 不应提交，但附近泄漏的指针可用于计算：

```python
libc_base = leaked_stderr - 0x2134a0
```

`0x2134a0` 是附件 `libc.so.6` 中 `_IO_2_1_stderr_` 的偏移。

### `0x100` 到 0 的截断

漏洞逻辑可简化为：

```c
unsigned int total = key_len + value_len;

if (total <= 0x100) {
    uint8_t alloc_len = total;
    void *p = malloc(alloc_len ? alloc_len : 1);

    memcpy(p, key, key_len);
    ((char *)p)[key_len] = '\0';
    memcpy((char *)p + key_len + 1, value, value_len);
}
```

选择：

```text
key_len   = 246
value_len = 10
total     = 256 = 0x100
```

则：

```text
检查使用的 total       = 256
实际分配使用的 uint8_t = 0
malloc 大小             = 1
后续写入                ≈ 257 字节
```

因此问题不是“缺少长度检查”，而是检查与分配使用了不同的数值域。宽类型上的合法值在窄化后发生环绕。

### 保持泄漏连接稳定堆布局

服务是多线程的。公开解法使用两个 TLS 连接：

1. 第一连接进入 `FOR_AGENT` 分支并取得 libc 泄漏；
2. 不关闭第一连接，让它停留在认证提示处；
3. 第二连接发送恶意 TLV，完成堆利用。

提前关闭泄漏连接会释放与线程相关的分配，改变相邻 chunk 和 arena 状态。保持连接打开既保留泄漏，也使第二线程的堆布局更接近本地复现结果。

服务在读取 client info 前还会泄漏输入缓冲区指针。现代 glibc tcache 使用 safe-linking：

```text
encoded_next = target_address ^ (chunk_address >> 12)
```

知道输入缓冲区与待污染 chunk 的位置后，就能伪造受保护的 `next` 指针，使后续同尺寸分配返回指定地址。

### 覆盖 `stdout` 而不是 GOT

Full RELRO 阻止 GOT 改写。利用目标改为 libc 中保存标准流对象地址的全局指针区域，附件版本中位于：

```python
stdio_pointer_area = libc_base + 0x213660
```

tcache poisoning 让一次分配落到该区域。保留 `stderr` 和 `stdin` 的原值，只把 `stdout` 改成攻击者输入缓冲区中伪造 `FILE` 的地址。

client info 解析结束后，认证失败路径会调用 `printf`。`printf` 通过 `stdout` 执行流操作，因此无需额外控制程序计数器，正常业务逻辑就会进入伪造对象。

### `_IO_wfile_jumps` 到 `setcontext`

伪造 `FILE` 使用 `_IO_wfile_jumps` 的宽字符调用链。关键关系是宽字符对象中的次级 vtable 会在偏移 `0x68` 处取函数指针。把相应结构指向 `setcontext`，可以让 glibc 以伪造对象附近的数据恢复寄存器。

恢复后的首个调用设置为：

```c
open("flag", O_RDONLY);
```

受控栈上的后续 ROP 完成：

```c
read(flag_fd, buffer, 0x100);
write(vpn_socket_fd, buffer, 0x100);
```

不能简单使用：

```text
system("sh<&6>&6")
```

服务通过 `accept4` 创建带 `SOCK_CLOEXEC` 的连接。`execve` 新 shell 时套接字会被关闭，命令即使执行也无法把结果送回客户端。ORW 留在原进程中，不跨越 `execve`，因而保留当前 TLS/VPN 连接。

### 远端偏移应从泄漏计算

本地与远端堆布局存在 `0x10` 差异。把攻击者缓冲区位置硬编码为 `arena + 常量` 会导致本地成功、远端失败。公开实例中稳定关系是：

```python
payload_base = leaked_input_buffer - 0x72c0
```

应优先使用“两个同一连接内地址的相对关系”，而不是假设 arena 后每次分配顺序完全一致。`0x72c0` 仍然与题目二进制和当前线程布局绑定，需要用本地容器验证。

### 利用顺序

完整主线为：

1. 建立 TLS 连接 A，发送 `FOR_AGENT` 记录，解析 `_IO_2_1_stderr_` 泄漏并计算 libc 基址。
2. 保持连接 A 停在认证阶段。
3. 建立连接 B，记录其输入缓冲区泄漏。
4. 发送 `key_len=246`、`value_len=10` 的 TLV，使分配长度截断为 0 并溢出相邻 free chunk。
5. 按 safe-linking 公式伪造 tcache `next`，让后续分配覆盖 libc 的标准流指针区域。
6. 把 `stdout` 改成伪造 `FILE`，其 wide-data/vtable 最终调用 `setcontext`。
7. 触发认证失败的 `printf`，恢复寄存器并执行 `open/read/write`。
8. 从当前 VPN 套接字收到：

```text
r3ctf{ThIs_i5_A-E@2y_waY_T0-coNTRol-hE@p-0N-dOU6I3_THRead0}
```

原始堆调试记录和具体伪造对象布局可参阅 [hax1ng 的 ezvpn 题解](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/pwn/ezvpn.md)。正文已经保留协议格式、截断条件、泄漏方式、两连接布局、safe-linking、FSOP、`setcontext` 与 ORW 的必要逻辑。

## 方法总结

- 核心技巧：让 `key_len + value_len` 在检查时等于 `0x100`、在 `uint8_t` 分配长度中变成 0，以小块大写制造堆溢出；再通过 tcache poisoning 和伪造 `FILE` 进入 `setcontext`。
- 识别信号：解析器若分别使用总长度、字段长度和窄类型缓存长度，应逐步跟踪每次提升、截断和补终止符后的真实写入量；“检查允许 256、字段只占 1 字节”是典型环绕信号。
- 复用要点：多线程服务中保留泄漏连接可稳定 arena；safe-linking 必须使用实际 chunk 地址编码；`SOCK_CLOEXEC` 会让基于 shell 的回连失效，此时应在原进程中直接 ORW。
