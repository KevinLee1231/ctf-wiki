# config

## 题目简述

程序的行为由 29 字节二进制配置 `config.dat` 决定。文件由小端序魔数、CRC32 和三段 TLV 数据组成；程序解析这些字段后，用循环异或恢复 flag 前半段，再根据四个校验值逐字符拼出后半段。关键不是爆破二进制，而是还原配置格式和各字段的语义。

## 解题过程

`chall.c` 先把文件开头解释为两个 32 位小端整数：

- 偏移 `0x00`：魔数 `0xDEADC0DE`；
- 偏移 `0x04`：CRC32。校验时先临时把该字段清零，再对整个文件计算 CRC；
- 偏移 `0x08` 起：重复出现 `type | length | value` 形式的 TLV。

发布文件中的三段 TLV 分别是：

```text
type 0x10, length 4:  1e 46 32 80          -> [30, 70, 50, 128]
type 0x20, length 7:  08 03 17 47 5a 54 26 -> 加密文本
type 0x30, length 4:  6b 33 79 21          -> b"k3y!"
```

类型 `0x20` 的值与 `k3y!` 循环异或，得到 `c0nf1g_`。类型 `0x10` 的四个值依次通过源码中的算术条件：`30+70=100`、`50*30=1500`、`128-70=58`、`30<<2=120`、`50/2=25`、`70>60`，对应字符 `r3v3rs`，程序最后再追加 `e`。

以下脚本直接解析 TLV；恢复 flag 不依赖重新计算 CRC，但若要伪造配置，则必须按程序的“CRC 字段清零”规则重算校验值。

```python
import struct

data = open("config.dat", "rb").read()
magic, stored_crc = struct.unpack_from("<II", data, 0)
assert magic == 0xDEADC0DE

fields = {}
offset = 8
while offset < len(data):
    field_type, length = data[offset], data[offset + 1]
    fields[field_type] = data[offset + 2:offset + 2 + length]
    offset += 2 + length

values = list(fields[0x10])
encrypted = fields[0x20]
key = fields[0x30]
part1 = bytes(b ^ key[i % len(key)] for i, b in enumerate(encrypted)).decode()

part2 = ""
part2 += "r" if values[0] + values[1] == 100 else ""
part2 += "3" if values[2] * values[0] == 1500 else ""
part2 += "v" if values[3] - values[1] == 58 else ""
part2 += "3" if values[0] << 2 == 120 else ""
part2 += "r" if values[2] // 2 == 25 else ""
part2 += "s" if values[1] > 60 else ""
part2 += "e"

print(f"tjctf{{{part1}{part2}}}")
```

输出为：

```text
tjctf{c0nf1g_r3v3rse}
```

## 方法总结

- 核心技巧：依据读取偏移和长度字段还原自定义 TLV，再分别处理异或数据与条件生成数据。
- 识别信号：固定魔数、长度驱动的指针递增，以及不同 `type` 分支通常意味着自定义容器格式。
- 复用要点：先明确端序和完整性校验覆盖范围；解析未知格式时始终检查 `offset + 2 + length` 是否越界。
