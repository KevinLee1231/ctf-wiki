# L3akCTF 2025 Pacman Writeup

## 题目简述

题目给出一个被自制 packer 处理过的 ELF。入口点不再指向原程序，而是落在新增的 `.pac` 节；原 `.text` 已经过三层可逆变换。解包后还能看到第二层校验：输入会经过 4 轮 Feistel 网络，再与 32 字节目标密文比较。

决定性障碍是分析自制加载器、恢复原始代码并逆转 Feistel，因此本文按 Reverse 归档。

## 解题过程

### 定位 `.pac` 加载器

用 `readelf -h` 和 `readelf -S` 检查文件，可以确认入口位于 `.pac`，而不是原始 `.text`。该节前部是位置无关的解密器，尾部嵌入了解包参数。设 `.pac` 节内容长度为 `loader_size`，则：

```text
loader_size - 0x24：跳回原入口的相对位移
loader_size - 0x20：16 字节 XXTEA key
loader_size - 0x10：8 字节 .text 虚拟地址
loader_size - 0x08：4 字节 .text 长度
```

这些值不是靠猜测得到的：加载器从 RIP 相对地址读取它们，先对目标页调用 `mprotect`，解密 `.text`，再跳回原入口。

### 逆转三层打包

打包器按以下顺序处理 `.text`：

1. 每字节先与 `0xAA` 异或，再加 `0x37`；
2. 每字节循环左移 3 位；
3. 对整个缓冲区执行 XXTEA。

解包时必须逆序执行：

```python
def ror8(x, n):
    return ((x >> n) | (x << (8 - n))) & 0xff

def undo_byte_layers(data):
    out = bytearray()
    for b in data:
        b = ror8(b, 3)
        b = ((b - 0x37) & 0xff) ^ 0xaa
        out.append(b)
    return out

plain_text = xxtea_decrypt(encrypted_text, key)
plain_text = undo_byte_layers(plain_text)
```

`xxtea_decrypt` 按标准 XXTEA 的逆循环实现，但字节序、分组为 32 位字以及尾块处理必须与加载器汇编一致。另一个稳妥办法是在程序跳回原入口前下断点：此时加载器已经在内存中完成全部解密，直接按记录的 `.text` 地址和长度转储即可。

若要重建可静态分析的 ELF，应通过节表或 `PT_LOAD` 映射把 `.text` 虚拟地址换算成文件偏移，再覆盖相应字节，不能把虚拟地址直接当作文件下标。

### 逆转 Feistel 校验

解包后的程序将 32 字节输入拆成两个 16 字节块。每块由两个小端 64 位整数组成，使用四个轮密钥：

```python
keys = [
    0x1337DEADBEEF,
    0xC0DE12345678,
    0xABCDEF012345,
    0x9876543210AB,
]

def f(x, key):
    x ^= key
    x = ((x << 13) | (x >> 51)) ^ (x * 31)
    return x & 0xffffffffffffffff
```

目标密文为：

```text
91 bc 04 8f 7a 48 83 fd 31 63 41 16 93 b2 a9 1e
4f 94 08 6b 54 a4 be 2f af dc 54 98 7e 9e 2e 92
```

Feistel 网络不要求轮函数可逆，只需按相反轮序恢复左右半部：

```python
import struct

def decrypt_block(block):
    left, right = struct.unpack("<QQ", block)
    for key in reversed(keys):
        old_left = left
        left = right ^ f(left, key)
        right = old_left
    return struct.pack("<QQ", left, right)

target = bytes.fromhex(
    "91bc048f7a4883fd3163411693b2a91e"
    "4f94086b54a4be2fafdc54987e9e2e92"
)
flag = decrypt_block(target[:16]) + decrypt_block(target[16:])
print(flag.decode())
```

输出为：

```text
L3AK{feistel_netWork_Is_fun!!!!}
```

## 方法总结

分析自制 packer 时，入口点、额外可执行节和加载器尾部数据比普通字符串更有价值。先明确解密目标、算法顺序和参数位置，再选择静态重建或运行时转储；后一种方法常能绕过实现 XXTEA、压缩或重定位细节的工作，但仍应理解加载器行为以确定正确的转储时机。

解包只是第一层。本题恢复出的原程序还包含 Feistel 校验，因此必须继续沿成功条件追踪。Feistel 的核心优势也是其逆向要点：轮函数本身无需求逆，只要交换左右半部并反序使用轮密钥即可恢复明文。
