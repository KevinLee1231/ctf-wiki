# low cortisol gachapon

## 题目简述

现场扭蛋机用 ESP32-C3 固件和 PN532 读取 MIFARE Classic 1K 卡。目标是制作一张能通过余额校验的卡，让固件驱动舵机。题目提供未剥离符号的固件和一份 1024 字节原始卡 dump，可从固件恢复认证密钥、数据块位置、记录格式和 CRC 校验。

## 解题过程

对 `gachapon` 执行基本静态检查，可以看到 `GREYGachaCard`、`Bad CRC`、`Insufficient balance`、`Card valid. Moving servo.` 等字符串，以及 `checkCardBalance`、`crc32` 等符号。固件使用默认运输密钥：

```text
FF FF FF FF FF FF
```

认证后读取 sector 4 的 block 16、17，并把两个 16 字节块拼成 32 字节记录。格式为：

| 偏移 | 长度 | 含义 |
| ---: | ---: | --- |
| 0 | 13 | ASCII `GREYGachaCard` |
| 13 | 1 | 版本 `0x01` |
| 14 | 4 | 余额，`uint32` 小端 |
| 18 | 4 | 前 18 字节的 CRC32，小端 |
| 22 | 10 | 忽略的填充 |

通过条件是余额至少为 670000 分。写成小端字节：

```text
670000 = 0x000A3930 -> 30 39 0A 00
```

对 `marker || version || balance` 共 18 字节计算标准 CRC32/IEEE，得到：

```text
CRC32 = 0x7D620DBA -> BA 0D 62 7D
```

因此两个目标块为：

```text
block 16:
47 52 45 59 47 61 63 68 61 43 61 72 64 01 30 39

block 17:
0A 00 BA 0D 62 7D 00 00 00 00 00 00 00 00 00 00
```

原始 MIFARE Classic 1K dump 每块 16 字节，所以 block 16 从文件偏移 `0x100` 开始。补丁逻辑为：

```python
head = b"GREYGachaCard" + b"\x01" + struct.pack("<I", 670000)
crc = struct.pack("<I", binascii.crc32(head) & 0xffffffff)
record = (head + crc).ljust(32, b"\x00")
dump[0x100:0x120] = record
```

将修改后的 1024 字节 dump 写回一张兼容的可写卡。以 Proxmark 和兼容 magic card 为例，可使用：

```text
hf mf cload -f patched.bin
```

把卡放到机器读卡区，串口会依次显示：

```text
Balance: 670000
Card valid. Moving servo.
```

舵机动作后完成现场挑战，flag 为：

```text
grey{i_l0ve_B4LLS!!}
```

## 方法总结

本题没有要求破解 MIFARE 密码算法：默认密钥允许正常认证，真正检查发生在应用层记录。只改余额而不重算 CRC 会失败；完整伪造必须同时满足 marker、版本、小端余额和 CRC32。先从固件恢复精确结构，再对副本做二进制补丁并校验块边界，能避免在有限的现场借用时间内反复试错。
