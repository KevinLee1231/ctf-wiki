# Shellcode

## 题目简述

附件是 Go 编译的文件加密器。程序在运行时把一段 Base64 数据解码为 x86-64 机器码，分配可执行内存后调用它，再遍历 `inputdir/` 中的文件按 8 字节分组加密。需要从 Go 的 `main.main` 中定位 shellcode，分析其分组算法，并解密生成的 `flag.enc`。

## 解题过程

### 从 Go 加载器提取机器码

Go 二进制符号较多，先定位 `main_main`。反编译结果中能看到 `encoding_base64__Encoding_DecodeString`、内存分配、`os_ReadFile`、间接调用和 `os_WriteFile`，据此可还原流程：

1. Base64 解码内置字符串。
2. 通过系统调用申请可执行内存并复制机器码。
3. 遍历 `inputdir/`，读取每个文件。
4. 每 8 字节调用一次解码后的函数。
5. 把结果写回输出文件。

内置字符串为：

```text
VUiD7FBIjWwkIEiJTUBIi0VAiwCJRQC4BAAAAEgDRUCLAIlFBMdFCAAAAADHRQwj782rx0UQFgAAAMdFFCEAAADHRRgsAAAAx0UcNwAAAMdFIAAAAACLRSCD+CBzWotFDANFCIlFCItFBMHgBANFEItVCANVBDPCi1UEweoFA1UUM8IDRQCJRQCLRQDB4AQDRRiLVQgDVQAzwotVAMHqBQNVHDPCA0UEiUUEuAEAAAADRSCJRSDrnkiLRUCLVQCJELgEAAAASANFQItVBIkQSI1lMF3D
```

可先独立解码，再以 64 位原始二进制方式载入 IDA：

```python
import base64

encoded = (
    "VUiD7FBIjWwkIEiJTUBIi0VAiwCJRQC4BAAAAEgDRUCLAIlFBMdFCAAAAADHRQwj"
    "782rx0UQFgAAAMdFFCEAAADHRRgsAAAAx0UcNwAAAMdFIAAAAACLRSCD+CBzWotF"
    "DANFCIlFCItFBMHgBANFEItVCANVBDPCi1UEweoFA1UUM8IDRQCJRQCLRQDB4AQD"
    "RRiLVQgDVQAzwotVAMHqBQNVHDPCA0UEiUUEuAEAAAADRSCJRSDrnkiLRUCLVQCJ"
    "ELgEAAAASANFQItVBIkQSI1lMF3D"
)

with open("shellcode.bin", "wb") as stream:
    stream.write(base64.b64decode(encoded))
```

### 识别魔改 TEA

机器码每次接收两个 32 位整数，共 64 位。核心循环执行 32 轮：

```c
uint32_t v0 = block[0];
uint32_t v1 = block[1];
uint32_t sum = 0;

for (int i = 0; i < 32; ++i) {
    sum -= 0x543210DD;
    v0 += ((v1 >> 5) + 33) ^ (v1 + sum) ^ ((v1 << 4) + 22);
    v1 += ((v0 >> 5) + 55) ^ (v0 + sum) ^ ((v0 << 4) + 44);
}
```

在模 $2^{32}$ 意义下，`-0x543210DD` 等价于 `+0xABCDEF23`。结构与 TEA 相同，只是轮常量和四个加数改成了 `0xABCDEF23` 与 `[22, 33, 44, 55]`。解密时必须反向更新 `v1`、`v0`，最后再回退轮和。

### 解密 `flag.enc`

Go 程序和 shellcode 都在 x86-64 上按小端序解释两个 `uint32`。下面的脚本显式保留 32 位溢出，并用 `struct` 避免整数转字节时丢失前导零：

```python
import struct

MASK = 0xFFFFFFFF
STEP = 0xABCDEF23


def decrypt_block(v0, v1):
    total = (STEP * 32) & MASK

    for _ in range(32):
        v1 = (
            v1
            - (((v0 >> 5) + 55) ^ (v0 + total) ^ ((v0 << 4) + 44))
        ) & MASK
        v0 = (
            v0
            - (((v1 >> 5) + 33) ^ (v1 + total) ^ ((v1 << 4) + 22))
        ) & MASK
        total = (total - STEP) & MASK

    return v0, v1


ciphertext = open("flag.enc", "rb").read()
assert len(ciphertext) % 8 == 0

plaintext = bytearray()
for v0, v1 in struct.iter_unpack("<2I", ciphertext):
    p0, p1 = decrypt_block(v0, v1)
    plaintext += struct.pack("<2I", p0, p1)

print(plaintext.rstrip(b"\x00"))
```

输出为：

```text
b"hgame{th1s_1s_th3_tutu's_h0mew0rk}"
```

官方 PDF 给出了加载器定位和加密函数，但没有转写 Base64 全串、完整解密代码及结果；这些缺口由参赛者的 [HGAME2023 逆向题解](https://oacia.dev/hgame2023-reverse-writeup/) 复核补全。

## 方法总结

分析运行时加载的 shellcode 时，应先把“加载器”和“实际算法”拆开：在宿主程序中确认数据来源、调用粒度与字节序，再把机器码导出为独立文件。识别出 TEA 类结构后，逆运算顺序、模 $2^{32}$ 溢出和端序缺一不可；只照搬标准 TEA 常量会得到完全错误的结果。
