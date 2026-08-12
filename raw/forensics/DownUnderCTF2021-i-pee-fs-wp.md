# DownUnderCTF 2021 - i-pee fs

## 题目简述

题目把一份 FAT16 磁盘镜像映射到一个 IPv6 前缀：向“前缀 + 单词索引”的目标地址发送 UDP 数据报，服务会取目标 IPv6 地址末 32 位作为索引，返回镜像中对应的一个 32 位小端整数。源码实际按远端地址的 `/56` 片段限速，每分钟最多响应 512 次有效请求，因此不能顺序下载整份 64 MiB 镜像。

目标是把这个稀疏字节读取接口当作远程磁盘，按 FAT 元数据只请求定位 flag 所需的扇区。

## 解题过程

每个远端请求对应四字节镜像数据。若要读取字节区间 `[offset, offset + length)`，先计算覆盖它的单词范围，再把每个十进制响应转回小端四字节：

```python
from ipaddress import ip_address
from pwn import p32
from socket import AF_INET6, SOCK_DGRAM, socket

BASE = ip_address("2600:1900:4120:5fb8::")

def get_word(sock, index):
    target = str(BASE + index)
    sock.sendto(b"", (target, 1337))
    packet, address = sock.recvfrom(64)
    if address[0] != target or int(packet) == -1:
        raise RuntimeError("request failed or rate limited")
    return p32(int(packet))
```

首先读取 FAT16 引导扇区中的 BIOS Parameter Block。关键字段为：

| 偏移 | 长度 | 含义 |
|---:|---:|---|
| `0x0b` | 2 | 每扇区字节数 `bytes_per_sector` |
| `0x0d` | 1 | 每簇扇区数 `sectors_per_cluster` |
| `0x0e` | 2 | 保留扇区数 `reserved_sectors` |
| `0x10` | 1 | FAT 表数量 `fat_count` |
| `0x11` | 2 | 根目录项数量 `root_entries` |
| `0x16` | 2 | 每张 FAT 的扇区数 `sectors_per_fat` |

于是可以直接定位 FAT、固定根目录和数据区：

```text
fat_start      = reserved_sectors * bytes_per_sector
root_start     = (reserved_sectors + fat_count * sectors_per_fat)
                 * bytes_per_sector
root_size      = ceil(root_entries * 32 / bytes_per_sector)
                 * bytes_per_sector
data_start     = root_start + root_size
```

FAT16 根目录项固定为 32 字节。逐项读取根目录即可，不必下载垃圾文件；短文件名在偏移 `0x00..0x07`，扩展名在 `0x08..0x0a`，首簇号位于 `0x1a`，文件大小位于 `0x1c`。找到名称含 `FLAG` 的项后，首簇 $n$ 的数据偏移为：

$$
\text{offset}(n)=\text{data\_start}+(n-2)\times\text{sectors\_per\_cluster}\times\text{bytes\_per\_sector}.
$$

flag 文件很小，读取首簇中由目录项给出的 `size` 字节即可；若文件跨簇，再按 FAT16 表中每项两字节的簇链继续读取。恢复出的内容为：

```text
You've made it! DUCTF{FAT32_0v3r_IPv16_q01KQYYS6a}
```

因此 flag 是：

```text
DUCTF{FAT32_0v3r_IPv16_q01KQYYS6a}
```

## 方法总结

网络协议只是镜像的随机访问传输层，决定性知识是 FAT16 的引导扇区、根目录项和簇寻址。面对限速的远程块设备，应先读取最小元数据，再定点请求目录与目标簇；不要把“能够枚举所有地址”等同于“应当暴力下载整个镜像”。官方脚本对固定镜像布局做了若干快捷计算，通用实现应显式读取 `reserved_sectors` 与 `root_entries`，以免参数变化后定位错误。
