# Pirated router

## 题目简述

附件是一份路由器固件镜像。解题重点不是网络服务或硬件接口，而是从固件文件系统中定位异常程序，并还原其内部的逐字节异或逻辑。固件中的大部分可执行文件是 MIPS 架构，`bin/secret_program` 却是 AArch64 ELF，这个架构差异正是定位目标的关键线索。

## 解题过程

先用 `binwalk` 递归解包固件，再检查解出的文件类型：

```bash
binwalk -Me firmware.bin
find _firmware.bin.extracted -type f -exec file {} \;
```

解包得到 SquashFS 根文件系统。逐项检查可执行文件时，可以发现 `bin/secret_program` 与其余 MIPS 程序架构不同：

```text
bin/secret_program: ELF 64-bit LSB executable, ARM aarch64, ...
```

在 IDA 中选择 AArch64 处理器打开该文件。关键逻辑先把 8 个 32 位整数写入局部数组，再将数组视为字节序列，每个字节与 `35`（即 `0x23`）异或后输出。官方简版题解没有列出这些常量；下列数值由 [HGAME2023 官方仓库收录的参赛者题解](https://github.com/vidar-team/HGAME2023_Writeup/blob/main/Week2/Non-HDUer_Writeups/Week2-vvbbnn00.pdf) 补齐：

```c
uint32_t data[8] = {
    0x4e42444b, 0x4d565846, 0x48401753, 0x7c444d12,
    0x4e514a45, 0x46514254, 0x7c50127c, 0x5a506210,
};

for (int i = 0; i <= 32; ++i) {
    putchar(((unsigned char *)data)[i] ^ 0x23);
}
```

整数来自小端程序，必须按小端序拆成连续字节。用 Python 复现有效的 32 字节常量区：

```python
import struct

words = [
    0x4E42444B,
    0x4D565846,
    0x48401753,
    0x7C444D12,
    0x4E514A45,
    0x46514254,
    0x7C50127C,
    0x5A506210,
]

ciphertext = struct.pack("<8I", *words)
decoded = bytes(byte ^ 0x23 for byte in ciphertext)
print(decoded.decode() + "}")
```

常量区实际解出：

```text
hgame{unp4ck1ng_firmware_1s_3Asy
```

反编译循环的条件是 `i <= 32`，但 `data[8]` 只有 32 字节，最后一次迭代会越界读取相邻栈数据，不能把这个未定义字节当成可靠的右花括号。结合固定的 flag 格式补全闭合符号，得到：

```text
hgame{unp4ck1ng_firmware_1s_3Asy}
```

## 方法总结

- 核心技巧：递归解包 SquashFS 固件，并用文件架构差异从大量系统程序中筛出可疑 ELF。
- 易错点：反编译器以 32 位整数展示常量时，需要按目标端序还原字节；同时不能忽略 `i <= 32` 对 32 字节数组造成的越界读取。
- 复用要点：分析固件时先建立文件类型与架构清单，再集中逆向异常文件；对解码结果与格式补全应分别说明，避免把未定义行为包装成确定结论。
