# Escape CET

## 题目简述

题目给出启用了 PIE、Full RELRO、Canary、NX、IBT 和 Shadow Stack 的 x86-64 程序。远端不是依赖宿主机原生 CET，而是通过 Intel SDE 启动：

```bash
sde64 -tgl -cet 1 -cet-endbr-exe 1 -- ./tcet
```

程序提供三个与题面“三件遗物”对应的原语：

- Ring：仅一次、仅限 libc 可写区的 4 字节写；
- Slab：在 libc 附近映射一块攻击者可写的大内存并泄漏地址；
- Bridge：经过完整性检查后执行一次可控间接调用。

Bridge 会检查 stdio vtable、`_IO_list_all`、退出处理器和 TCB 等敏感状态，常见的 FSOP、退出处理器覆盖和直接返回链都受到限制。真正的路线是借助 Ring 改写 libc locale 指针，在 Slab 中伪造 locale、gconv、obstack 和 `ucontext_t`，让一次合法的 IBT 间接调用最终修改线程本地 CET 状态、进入 `setcontext`，再调用 `arch_prctl` 关闭 Shadow Stack，之后使用 SROP 完成任意系统调用。

容器中的 `/flag` 不是明文 flag，而是 AES-256-CBC 密文说明。每次连接都会重新生成 32 字节 key，并把真实 flag 加密后写入 `/flag`。输出层还会把连续出现的完整 key 替换成星号，因此利用完成任意文件读后仍需绕过输出过滤。

## 解题过程

### 三个原语与地址关系

Slab 初始化后会打印映射地址和跨度。对于题目随附的 libc，公开解法使用：

```python
libc_base = slab_addr + 0x10000000 + slab_span + 0xa000
```

这个关系由程序的定址 `mmap` 方式产生，不应套用到重新编译的二进制。Slab 用于保存伪造对象、上下文、SROP frame、路径和读写缓冲区。

Ring 接收一个 libc 可写区地址和 32 位值，可视为：

```c
*(uint32_t *)address = value;
```

Ring 使用后才会解锁 Bridge。Bridge 的函数指针只能设置一次，并要求目标满足 IBT；因此最终触发点选择 libc 中以合法入口形式存在的 `getwchar`，而不是任意中间 gadget。

题目 libc 中公开解法使用的主要偏移为：

```python
GETW           = 0x8ff80
LOCALE_PTR     = 0x213380
OBSTACK_NEW    = 0xb6ee0
H_ERRNO_LOC    = 0x146b00
SETCONTEXT     = 0x4be80
ARCH_PRCTL     = 0x138480
POP_RAX        = 0xe5e47
SYSCALL_RET    = 0xa0c46
```

这些偏移必须针对附件的 `libc.so.6` 使用；更新 libc 后需要重新定位。

### 从 locale/gconv 路径写入 TLS

Ring 的唯一一次写用于把 libc 中的 locale 指针低 32 位改到 Slab 内的伪造 locale。Slab 中继续布置：

```text
fake locale
  -> fake gconv step/data
  -> fake obstack
  -> controlled copy source and destination
  -> fake ucontext
```

Bridge 设置为 `getwchar` 后，向标准输入再发送一个字节，宽字符转换路径就会沿伪造的 locale/gconv 对象执行。伪造 obstack 利用 `_obstack_newchunk` 的复制和回调行为，把如下模式写入线程本地存储中与 CET 功能状态有关的位置：

```python
TLS_CET_PATTERN = 0x0000000200000002
```

这一步的目的不是直接破坏返回地址，而是让 glibc 的 `setcontext` 走到可以恢复攻击者上下文的路径。直接按照“原生硬件 CET 已正确初始化”的假设构造上下文在远端会失败，因为 SDE 环境下 glibc 的 CET bookkeeping 与原生环境不同。

### `setcontext` 后关闭 CET

伪造上下文首先调用：

```c
arch_prctl(ARCH_CET_DISABLE, 2);
```

其中：

```python
ARCH_CET_DISABLE = 0x3002
```

第二个参数的位 `2` 表示关闭 Shadow Stack。IBT 约束下的第一阶段只负责安全到达这里；Shadow Stack 关闭后，Slab 中预先布置的伪造信号帧即可通过 `rt_sigreturn` 批量恢复寄存器。

后续 SROP 链执行：

```text
openat2(AT_FDCWD, path, &how, sizeof(how))
read(fd, buffer, size)
write(1, buffer, size)
```

Intel SDE 和包装进程已经占用了多个文件描述符，公开环境中新文件落在 fd 10，而不是常见的 fd 3。稳妥实现应先枚举或在本地复现环境中确认，不要把 fd 3 当成 ABI 保证。

Bridge 的最终触发形态为：

```python
send(b"A" * 0x20 + p64(libc_base + GETW))
send(b"Z")
```

第一行设置间接调用目标，第二行给宽字符转换路径提供输入，从而启动 locale/gconv、obstack、`setcontext`、`arch_prctl` 和 SROP 的整条链。

### `/flag` 为什么不是最终结果

本地 `start.sh` 明确执行：

1. 从 `/root/flag` 读取真实 flag；
2. 生成 32 个字母数字字符作为随机 key；
3. 调用 `/home/ctf/encryptor`；
4. 把 Base64 密文和 key 路径写入公开的 `/flag`。

逆向 `encryptor` 可确认算法为：

```text
AES-256-CBC
IV = 16 个零字节
PKCS#7 padding
```

直接读取 `/home/ctf/key` 会看到 32 个 `*`，但这不是实际 key。服务的输出过滤器会查找完整连续 key 并替换它，因此用星号作为 AES key 解密必然失败。

### 把 key 拆散输出

任意系统调用已经可用后，使用 `writev` 把 key 的每个字节作为独立 iovec，并在相邻字节间插入换行：

```text
k[0]\n
k[1]\n
...
k[31]\n
```

输出流中不再出现连续的 32 字节 key，过滤器无法匹配整体秘密。客户端删除分隔换行，重新拼出本连接的实际 key。

随后在同一连接中读取 `/flag`，解析其中的 Base64 密文并解密：

```python
import base64
from Crypto.Cipher import AES

def unpad_pkcs7(data):
    pad = data[-1]
    if not 1 <= pad <= 16 or data[-pad:] != bytes([pad]) * pad:
        raise ValueError("invalid PKCS#7 padding")
    return data[:-pad]

key = recovered_key
ciphertext = base64.b64decode(ciphertext_b64)
plaintext = AES.new(
    key,
    AES.MODE_CBC,
    iv=b"\x00" * 16,
).decrypt(ciphertext)
print(unpad_pkcs7(plaintext).decode())
```

key 和密文都是每连接生成的，不能在一次连接中泄漏 key、换到另一实例再读取密文。成功结果为：

```text
r3ctf{coNGRaTS_YoU-hElp3D_cR@Zym4N-P4SS_c3T_WlTH-sHAD0W_Stack_and_ibt🎉0}
```

完整的伪造对象布局与原始运行记录可参阅 [hax1ng 的 Escape CET 题解](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/pwn/Escape-CET.md)。正文已经归纳 Ring/Slab/Bridge 的作用、SDE 与原生 CET 的差异、locale/gconv 到 TLS 的写入链、CET 关闭、SROP 和 key 过滤绕过，外链不承担必要步骤。

## 方法总结

- 核心技巧：用受限 libc 写把正常的 locale 入口重定向到伪造对象，经 gconv/obstack 获得 TLS 写入；在仍受 IBT/Shadow Stack 约束时进入 `setcontext`，调用 `arch_prctl` 关闭 CET，再使用 SROP 做文件读取。
- 识别信号：当目标给出“少量数据写 + 邻近可控映射 + 单次受检间接调用”，且远端在模拟器中启用 CET 时，应分别审计应用完整性检查、glibc 数据驱动调用链和模拟器实际维护的 CET 状态。
- 复用要点：不要把原生 CET 行为、fd 3 或 libc 偏移当成固定事实；拿到任意读后还要检查服务包装层是否加密或过滤输出；涉及每连接随机材料时，泄漏、密文读取和解密必须在同一实例完成。
