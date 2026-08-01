# GlacierCTF 2024 Ramo

## 题目简述

题目给出 Windows x64 加密程序 `Ramo.exe` 和 80 字节的 `flag.txt.enc`。程序用当前时间与明文哈希的异或值作为自定义伪随机数生成器种子，据此生成 AES key 与 IV，再以 CBC 模式加密 64 字节 flag。输出中每个 16 字节密文块前还附带 4 字节校验值，恰好泄露种子的一个字节。

官方 WP 为空，但仓库同时保存了加密、解密源码和最终二进制。需要特别注意：源码注释与 `main.cpp` 都称其为 AES-256，实际构建却因预处理宏只在单个编译单元生效而使用 AES-128。若机械照抄注释，就会得到正确种子、key 和 IV，却始终无法解密。

## 解题过程

### 1. 还原输出格式与种子泄露

加密前先计算：

```c
uint32_t t = (uint32_t)time(NULL);
t ^= hash(input_data_size, input_data);
seed = t;
```

随后每个密文块按以下格式写出：

```text
[hash(t 的第 i 个字节, "flag.txt")：4 字节小端]
[AES-CBC 密文：16 字节]
```

四组记录共 $4\times(4+16)=80$ 字节。哈希是带可控初值的 djb2：

```python
def djb2(initial, data=b"flag.txt"):
    h = (5381 + initial) & 0xffffffff
    for c in data:
        h = (h * 33 + c) & 0xffffffff
    return h
```

对每个记录枚举 `0..255`，找到满足目标哈希的字节即可重组 32 位小端种子：

```python
from pathlib import Path
import struct

blob = Path("flag.txt.enc").read_bytes()
seed_bytes = []
ciphertext = bytearray()

for off in range(0, len(blob), 20):
    target = struct.unpack_from("<I", blob, off)[0]
    seed_bytes.append(next(k for k in range(256) if djb2(k) == target))
    ciphertext += blob[off + 4:off + 20]

seed = int.from_bytes(seed_bytes, "little")
print(seed_bytes, hex(seed))
```

仓库样本恢复出：

```text
seed bytes = 0x12 0x52 0x75 0x55
seed       = 0x55755212
```

### 2. 精确复现自定义 rand

程序没有使用 C 运行库 `rand()`，而是自行实现了 MSVC 风格的线性同余生成器：

```python
state = 0x55755212

def custom_rand():
    global state
    state = (state * 0x343fd + 0x269ec3) & 0xffffffff
    return (state >> 16) & 0x7fff

def random_string(n):
    return bytes(0x20 + custom_rand() % (0x7e - 0x20) for _ in range(n))

key32 = random_string(32)
iv = random_string(16)
```

结果为：

```text
key32 = Unk]n8&o7>AZFNyNAmxK=U1$t+(EL/D}
iv    = P6QwgkxeHKYzb:Gk
```

### 3. 识别实际使用的是 AES-128

`main.cpp` 在包含头文件前定义 `AES256`，但 `aes.cpp` 是另一个编译单元，它只看到 `aes.hpp` 中默认启用的：

```c
#define AES128 1
//#define AES256 1
```

因此 AES 实现文件按 `Nk=4`、`Nr=10` 编译，密钥扩展只读取前 16 字节。`AES_ctx` 的 IV 偏移也按 AES-128 版本解释；加密和解密函数都来自这个编译单元，所以最终行为稳定地是 AES-128-CBC，而不是注释所称的 AES-256-CBC。

用前 16 字节 key 解密即可：

```python
from Crypto.Cipher import AES

plaintext = AES.new(key32[:16], AES.MODE_CBC, iv).decrypt(bytes(ciphertext))
print(plaintext.decode())
```

输出为：

```text
gctf{i_h0p3_y0u_used_th3_c0rrect_r4nd0m_funct10n_s0lv1Ng_th1s!!}
```

该结果已使用仓库中的 `flag.txt.enc` 本地解密验证，并与随源码保存的 64 字节明文一致。

## 方法总结

利用链是“块前哈希枚举恢复 4 字节种子 → 精确复现自定义 LCG → 生成 key/IV → 依据最终构建而非注释确定 AES 轮数”。本题最容易遗漏的是 C/C++ 预处理宏只作用于当前编译单元：`main.cpp` 里的 `#define AES256` 不会自动传给单独编译的 `aes.cpp`。逆向题应以二进制行为、项目构建方式和可重复解密结果交叉验证源码注释。
