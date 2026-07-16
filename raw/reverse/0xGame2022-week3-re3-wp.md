# week3re3

## 题目简述

程序在 `.text` 中插入多组互补条件跳转和异常 `call` 字节，破坏反汇编器的控制流识别。清理花指令后，核心逻辑是以 `0xGame2022` 为密钥的标准 RC4；内置密文经同一 RC4 流再次异或即可恢复 flag。

## 解题过程

在 IDA 中观察到三种重复字节模式。前两种利用 `jz`/`jnz` 互补分支制造重叠反汇编，第三种用短 `call` 和栈相关字节扰乱函数边界。确认这些序列不参与有效计算后，可在副本上批量替换为等长 NOP：

```python
import ida_bytes

start = 0x401000
end = 0x401908

patterns = (
    b"\x31\xC0\x74\x03\x75\x01\xE8",
    b"\x74\x04\x75\x02\xE8\xE8",
    b"\xE8\x01\x00\x00\x00\x83\x83\x04\x24\x04\xC3\xF3",
)

for address in range(start, end):
    for pattern in patterns:
        if ida_bytes.get_bytes(address, len(pattern)) == pattern:
            print(hex(address), pattern.hex())
            ida_bytes.patch_bytes(address, b"\x90" * len(pattern))
```

重新分析函数后可以看到 256 字节状态数组的初始化与交换循环，即 RC4 的 KSA；后续逐字节更新 `i`、`j` 并异或 `S[(S[i]+S[j]) mod 256]`，对应 PRGA。提取密钥和密文后直接复现：

```python
key = b"0xGame2022"
cipher = bytes([
    0xDD, 0x84, 0x73, 0x53, 0xEC, 0x7C, 0x5C, 0xD4,
    0xE6, 0x5E, 0xE2, 0x43, 0x5F, 0x39, 0xE3, 0x6E,
    0x5E, 0x8E, 0x11, 0x97, 0xFC, 0x24, 0x49, 0x60,
    0x77, 0x29, 0x10, 0xDA, 0x23, 0x4D, 0x3D, 0x38,
    0x0A, 0x76, 0xB6, 0x08, 0xC0, 0x38, 0x91, 0x28,
    0x43, 0x91,
])

# KSA
sbox = list(range(256))
j = 0
for i in range(256):
    j = (j + sbox[i] + key[i % len(key)]) & 0xFF
    sbox[i], sbox[j] = sbox[j], sbox[i]

# PRGA
i = j = 0
plain = bytearray()
for value in cipher:
    i = (i + 1) & 0xFF
    j = (j + sbox[i]) & 0xFF
    sbox[i], sbox[j] = sbox[j], sbox[i]
    stream_byte = sbox[(sbox[i] + sbox[j]) & 0xFF]
    plain.append(value ^ stream_byte)

print(plain.decode())
```

输出：

```text
flag{bc3ed83b-ffe3-4122-8fbe-651b1a591da7}
```

## 方法总结

处理花指令时应先识别重复模式和真实可达路径，再对文件副本做等长 NOP，不能仅因反汇编失败就删除任意字节。算法识别则依赖 RC4 的两阶段特征：KSA 初始化并置换 256 字节数组，PRGA 每轮交换两个元素并生成一个密钥流字节。RC4 加解密完全相同，因此提取密钥与密文后无需逆向另一套解密函数。
