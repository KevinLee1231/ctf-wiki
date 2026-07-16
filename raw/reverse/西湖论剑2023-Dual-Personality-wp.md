# Dual Personality

## 题目简述

题目给出一个 32 位 Windows 控制台程序，输入正确 32 字节字符串时输出 `Right, flag is DASCTF{your input}`。程序导入 `VirtualAlloc`、`VirtualProtect`，并在运行时把返回地址 patch 成 `EA <addr> 33 00` 形式的 far jump，借 WoW64 的 `0x33` 代码段执行 64 位代码。

核心机制是 Heaven's Gate 混合位宽执行：同一段字节在 32 位视角下像无意义指令，在 64 位视角下才是真正的反调试、旋转和 XOR 逻辑。

## 解题过程

### 关键观察

程序主逻辑读取输入到 `0x407060`，最后执行：

```text
memcmp(transformed_input, cipher, 0x20)
```

硬编码 32 字节 `cipher` 为：

```text
aa 4f 0f e2 e4 41 99 54 2c 2b 84 7e bc 8f 8b 78
d3 73 88 5e ae 47 85 70 31 b3 09 ce 13 f5 0d ca
```

实际变换分三层：

```text
input
  -> 8 个 little-endian dword 的 rolling add/xor
  -> 4 个 little-endian qword 的 rol64
  -> 循环 xor 04 77 82 4a
  -> memcmp(cipher)
```

第一段 64 位代码读取 `PEB.BeingDebugged`。非调试环境下写入 `0x5df966ae`，随后 32 位逻辑减去 `0x21524111`，得到 rolling 初始值：

```text
rolling = 0x3ca7259d
```

第三段 64 位代码由初始字节 `9d 44 37 b5` 生成 XOR key：

```text
04 77 82 4a
```

### 求解步骤

逆向顺序为：先 XOR，再对 4 个 qword 做 `ror64`，最后逆 rolling dword 链。关键脚本如下：

```python
import struct

MASK32 = 0xffffffff
MASK64 = 0xffffffffffffffff

def ror64(x, r):
    r %= 64
    return ((x >> r) | (x << (64 - r))) & MASK64

cipher = bytes.fromhex(
    "aa4f0fe2e44199542c2b847ebc8f8b78"
    "d373885eae47857031b309ce13f50dca"
)

xor_key = bytes([0x9d & 0x44, 0x44 | 0x37, 0x37 ^ 0xb5, (~0xb5) & 0xff])
buf = bytes(b ^ xor_key[i % 4] for i, b in enumerate(cipher))

after_chain = bytearray()
for i, rot in enumerate([12, 34, 56, 14]):
    q = struct.unpack_from("<Q", buf, i * 8)[0]
    after_chain += struct.pack("<Q", ror64(q, rot))

rolling = (0x5df966ae - 0x21524111) & MASK32
plain = bytearray()
for i in range(8):
    transformed = struct.unpack_from("<I", after_chain, i * 4)[0]
    original = (transformed - rolling) & MASK32
    plain += struct.pack("<I", original)
    rolling ^= transformed

inner = plain.decode("ascii")
print(f"DASCTF{{{inner}}}")
```

输出：

```text
DASCTF{6cc1e44811647d38a15017e389b3f704}
```

## 方法总结

- 核心技巧：识别 Heaven's Gate far jump，把 32 位 PE 中的混合字节按 64 位重新反汇编。
- 识别信号：PE32 中出现 `EA ... 33 00`、大量在 32 位模式下不合理的 `0x48` 前缀、`VirtualProtect` 修改代码时，要考虑 WoW64 混合执行。
- 复用要点：rolling key 逆向时更新仍应使用正向变换后的 dword；求出输入后要正向复算到 `cipher`，避免被反调试分支误导。
