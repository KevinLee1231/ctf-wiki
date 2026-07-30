# Missing Piece

## 题目简述

附件是一份无法正常挂载的磁盘镜像。镜像中存在已删除的工具和密码文件，磁盘尾部还有一段未登记在当前 MBR 中的数据。目标是恢复删除文件、还原加密程序的行为，并解密被隐藏的分区。

## 解题过程

先用 The Sleuth Kit 查看分区表：

```text
$ mmls disk.img
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

Slot      Start        End          Length       Description
Meta      0000000000   0000000000   0000000001   Primary Table
          0000000000   0000002047   0000002048   Unallocated
000:000   0000002048   0002078719   0002076672   Win95 FAT32
          0002078720   0002099199   0000020480   Unallocated
```

可见 FAT32 分区从 2048 扇区开始，长度为 2076672 扇区；紧随其后的 20480 个未分配扇区恰好是 10 MiB，很像被抹去分区表项后留下的隐藏分区。

列出并恢复 FAT32 中的删除文件：

```text
$ fls -o 2048 disk.img
r/r * 6: cryptodisko.passwd
r/r * 8: cryptodisko

$ mkdir recovered
$ tsk_recover -o 2048 disk.img recovered
```

`cryptodisko.passwd` 中保存了 25 字节字符串：

```text
PwnT0p14_s3cr3t_p4rt1t10n
```

`cryptodisko` 是保留符号的 Go ELF。逆向其分区加密函数可以还原以下流程：

1. 选择最后一个分区；
2. 把该分区的 16 字节 MBR 表项同时作为 AES-CBC 的 IV；
3. 将不足 32 字节的密钥补齐；本程序重复原始密钥长度 `0x19`，因此末尾是 7 个 `0x19`；
4. 使用 AES-256-CBC 加密分区内容；
5. 在密文前写入 16 字节 IV；
6. 把 MBR 中对应的分区表项清零。

当前可见分区的结束位置是：

$$
2048+2076672=2078720
$$

直接读取该扇区的前 16 字节可得到：

```text
000000000b00000000b81f0000500000
```

将其按 MBR 表项解释，分区类型为 `0x0b`，起始 LBA 为 `2078720`，长度为 `20480` 扇区。这同时证明了尾部未分配区就是被隐藏的原分区。正确的解密脚本如下：

```python
import struct
from pathlib import Path

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

SECTOR_SIZE = 512
VISIBLE_START = 2048
VISIBLE_LENGTH = 2076672
HIDDEN_START = VISIBLE_START + VISIBLE_LENGTH

offset = HIDDEN_START * SECTOR_SIZE

with open("disk.img", "rb") as disk:
    disk.seek(offset)
    iv = disk.read(16)
    ciphertext = disk.read()

partition_start = struct.unpack("<I", iv[8:12])[0]
partition_sectors = struct.unpack("<I", iv[12:16])[0]
assert partition_start == HIDDEN_START
assert partition_sectors == 20480

key = (
    b"PwnT0p14_s3cr3t_p4rt1t10n"
    + b"\x19" * 7
)
plain = unpad(
    AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext),
    AES.block_size,
)
Path("hidden-partition.img").write_bytes(plain)

begin = plain.index(b"N0PS{")
end = plain.index(b"}", begin) + 1
print(plain[begin:end].decode())
```

这里应读取 IV 后直到镜像末尾的全部 10485776 字节密文；解密后最后一个完整 AES 块是 16 字节填充，去除后正好得到 10 MiB 分区内容。运行结果为：

```text
N0PS{mBr_t4bl3_h4s_n0_s3cr3t_4_U_4Nym0r3}
```

仓库原题解中的示例脚本从 `0x1BE` 处读取当前第一个分区的起始 LBA，因而会错误地得到 `2048`。这里根据 `mmls` 边界、尾部 IV 中保存的原分区表项以及实际解密结果，将起点修正为 `2078720`。

## 方法总结

本题把磁盘取证、已删除文件恢复和二进制逆向串在一起。关键证据链是：可见分区后的 10 MiB 空洞、恢复出的加密工具与口令、尾部数据开头保存的原 MBR 表项，以及能够通过 PKCS#7 校验并检出 flag 的解密结果。磁盘题中的“未分配”只表示当前元数据没有引用该区域，并不代表其中没有有效内容；计算偏移时也应同时用分区边界和原始字节相互验证。
