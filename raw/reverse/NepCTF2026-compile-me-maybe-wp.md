# NepCTF2026 compile_me_maybe Writeup

## 题目简述

附件看似要求直接编译 C++ 项目，但 `include/generated_witness.hpp` 实际上是一个 FIFO（命名管道）。编译器处理 `#include` 时会一直等待写端，因此“编译很慢”只是表象；真正任务是从公开的 `main.cpp`、`vm.hpp` 和 `data.hpp` 中恢复缺失的 64 字节 `witness`。

程序会在编译期把 witness 按小端序拆成 16 个 `uint32_t` 寄存器，执行 982 字节的自定义 VM 程序，并与 `checker_target` 比较。解析后共有 261 条指令，最后一条为 `HALT`。

## 解题过程

先确认阻塞点：

```bash
find . -maxdepth 3 -ls
file include/generated_witness.hpp
```

需要补出的头文件类型如下：

```cpp
namespace cmc {
using witness = witness_bytes</* 64 bytes */>;
}
```

VM 的 opcode 还经过逐指令掩码。第 `step` 条指令的真实 opcode 为：

```python
MASK32 = 0xffffffff

def opcode_mask(step):
    value = (0xA5F1523D + step * 0x9E3779B9) & MASK32
    value ^= value >> 16
    value = (value * 0x7FEB352D) & MASK32
    value ^= value >> 15
    value = (value * 0x846CA68B) & MASK32
    value ^= value >> 16
    return value & 0xff

opcode = code[pc] ^ opcode_mask(step)
```

解析时只有 opcode 参与掩码，`XORI`、`ADDI`、`MULI` 的 32 位立即数仍按小端序读取。主要指令包括异或、加法、循环移位、寄存器交换、半字节 S-box、寄存器置换、奇数乘法和双寄存器 `MIX`。这些操作在 $2^{32}$ 模环上都是双射，因此不必爆破，可以从 `checker_target` 倒放整个程序：

- 异或、按位取反和交换均为自逆；
- 加法改为减法，循环左移改为循环右移；
- 奇数乘法使用模 $2^{32}$ 的乘法逆元；
- S-box 和置换表预先构造逆表；
- 所有中间结果都截断到 32 位。

`MIX` 的正向关系为：

```text
a' = a + rotl32(b, amount)
b' = b XOR rotl32(a', amount * 3 + 1)
```

逆运算必须先恢复 `b`，再恢复 `a`：

```python
b = b_prime ^ rotl32(a_prime, amount * 3 + 1)
a = (a_prime - rotl32(b, amount)) & 0xffffffff
```

完整倒放后得到 64 字节 witness：

```text
d4b882db1aa7e5ded432dfdf4cdb0c41cf6d0ace92d4e41acc6889fe6fff7868
12149aead035e4aba29cff4ef3f4477ff9588671474433979372275d658590a1
```

将其写成编译期类型即可：

```cpp
#pragma once

namespace cmc {
using witness = witness_bytes<
    0xd4, 0xb8, 0x82, 0xdb, 0x1a, 0xa7, 0xe5, 0xde,
    0xd4, 0x32, 0xdf, 0xdf, 0x4c, 0xdb, 0x0c, 0x41,
    0xcf, 0x6d, 0x0a, 0xce, 0x92, 0xd4, 0xe4, 0x1a,
    0xcc, 0x68, 0x89, 0xfe, 0x6f, 0xff, 0x78, 0x68,
    0x12, 0x14, 0x9a, 0xea, 0xd0, 0x35, 0xe4, 0xab,
    0xa2, 0x9c, 0xff, 0x4e, 0xf3, 0xf4, 0x47, 0x7f,
    0xf9, 0x58, 0x86, 0x71, 0x47, 0x44, 0x33, 0x97,
    0x93, 0x72, 0x27, 0x5d, 0x65, 0x85, 0x90, 0xa1
>;
}
```

向 FIFO 写入上述内容，或用普通文件替换它，再执行 `make run`，编译期校验通过后即可得到 flag。

## 方法总结

本题的关键不是等待编译，而是识别 `generated_witness.hpp` 的 FIFO 属性，并把编译期模板校验还原为可逆 VM。面对这类题，应先检查异常文件类型和构建流程，再判断校验指令是否逐步可逆。尤其要严格复现无符号 32 位溢出、指令计数和 `MIX` 的逆序依赖，否则即使大部分寄存器正确，最终 witness 仍无法通过编译期断言。
