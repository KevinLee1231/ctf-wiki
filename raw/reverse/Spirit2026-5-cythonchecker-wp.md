# CythonChecker WriteUp

## 题目简述

题目主程序是 `loader.exe`，它本身不直接校验 flag，而是嵌入 Python 运行时并动态释放 Cython 扩展 `runtime_codec.pyd`。loader 运行时会恢复两把 16 字节 key：一把作为 XXTEA key 解出 `.pyd`，另一把作为参数传给扩展中的 `verify_flag(flag, key_bytes)`。

核心机制分成两层：外层 loader 负责恢复 key、释放 `.pyd` 并调用 Python 扩展；真正校验逻辑在 Cython/Python 扩展里。扩展实现了 AES-family 结构，但 ShiftRows 被改动，需要按实际变体逆出目标密文对应的明文 payload，再拼回 `SpiritCTF{...}` 格式。

## 解题过程

CythonChecker WriteUp

题目概览

附件主程序是 loader.exe 。程序本身不直接做 flag 校验，而是：
1. 在运行时恢复两把 16 字节 key
2. 用其中一把 key 解出并释放 runtime_codec.pyd
3. 初始化嵌入式 Python 3.12
4. import runtime_codec ，调用其中唯一暴露的 verify_flag(flag, key_bytes)
这题的核心是：
从 loader.exe  里恢复真实 key
拿到真正参与校验的 .pyd
识别 .pyd  里的 AES-family 逻辑
找到被改动的 ShiftRows
逆出目标密文对应的明文 payload
最终 flag 为：
1. 先看 loader.exe

把 dist/challenge/loader.exe  挂进 IDA，先看字符串和导入。
可以立刻看到几个关键字符串：
runtime_codec
verify_flag
Flag:
Correct
Wrong
同时导入表里有大量 python312.dll  的 API，例如：
PyImport_ImportModule
PyObject_GetAttrString
PyBytes_FromStringAndSize
PyObject_CallFunctionObjArgs
Py_InitializeFromConfig
这说明程序不是自己校验，而是加载一个 Python 扩展模块做真正的验证。
SpiritCTF{Sh1fT_r0w3_aRe_NoT_wHa7_Y0u_eXp3cT}


1.1 主流程

main  位于：
从反编译可以分出三段关键逻辑：
sub_140008860 ：解密并释放 runtime_codec.pyd
sub_140009520 ：初始化嵌入式 Python 运行时
sub_140009E20 ：导入 runtime_codec  并调用 verify_flag
sub_140009E20  非常直白，核心就是：
所以这题的做法已经基本明确：先把外层恢复出来，再去分析 .pyd 。
2. 恢复 loader 中的两把 key

恢复 key 的函数在：
这个函数不复杂，本质上是一个按字节的线性异或恢复：
它会被调用两次：
unk_14002C980 ：恢复传给 verify_flag  的 AES key
unk_14002C998 ：恢复解包 .pyd  的 XXTEA key
恢复结果如下：
可以直接写个极短脚本恢复：
0x14000BB50

```c
PyImport_ImportModule("runtime_codec");
PyObject_GetAttrString(mod, "verify_flag");
PyUnicode_FromStringAndSize(flag, ...);
PyBytes_FromStringAndSize(aes_key, 16);
PyObject_CallFunctionObjArgs(verify_flag, flag_obj, key_obj, NULL);
```

0x140003660

```python
real[i] = obf[i] ^ ((seed + step * i) & 0xff)
```

AES key   = 1145141919810abc1145141919810cba
XXTEA key = eac1331ea1234567eac1331ea7654321


3. 释放并拿到真正的 runtime_codec.pyd

sub_140008860 @ 0x140008860  会做三件事：
1. 调上面的恢复函数拿到 XXTEA key
2. 从 unk_14002C9B0  拷出一大段加密 blob
3. XXTEA 解密后写到临时目录，文件名就是 runtime_codec.pyd
所以拿 .pyd  有两种办法：
最省事：运行一次程序，直接从 %TEMP%\\spiritwarm_runtime_*\\runtime_codec.pyd  复制出来
更完整：静态恢复 XXTEA key，自行解密 blob
XXTEA 部分是标准实现，解完以后可以得到真正的 .pyd 。
这里有一个很重要的注意点：
后续分析必须使用 loader.exe  实际释放出来的那份 .pyd 。
如果你误用了工程目录里别的旧构建文件，target_ciphertext  和当前外壳里的 key 可能对不上。
4. 分析 runtime_codec.pyd

把刚刚提取出来的 .pyd  再挂进 IDA。
4.1 第一眼特征

字符串里直接能看到：
_substitute
_mix_columns
_expand_round_keys
_variant_shift_rows
_encrypt_ecb
target_block_count
target_ciphertext
这已经很像 AES 了，至少说明它不是自定义杂糅算法，而是标准结构上做了轻微改动。

```python
def decode(blob18: bytes) -> bytes:
    data = blob18[:16]
    seed = blob18[16]
    step = blob18[17]
    return bytes(data[i] ^ ((seed + step * i) & 0xff) for i in range(16))
```


4.2 真正的校验函数

模块初始化入口是：
对外暴露的 verify_flag  包装在：
真正干活的函数体在：
从 0x180007690  的反编译里可以直接读出完整的校验逻辑：
1. flag  必须是 str
2. key_bytes  必须是 bytes  或 bytearray
3. flag 必须满足 SpiritCTF{...}
4. 取出 payload，长度必须在 1..64
5. payload 必须能按 ASCII 编码，且必须是 printable ASCII
6. 对 payload 做 PKCS#7
7. 调 _encrypt_ecb(padded, key_bytes)
8. 将结果与 target_ciphertext  逐字节比较
到这里基本可以确定：题目核心就是“拿到正确 key 后，逆 AES 变种的目标密文”。
5. 识别 AES 变种点

继续跟 _variant_shift_rows 。
相关包装函数在：
真正的实现核心在：
从逻辑可以看出它不是标准 AES 的 ShiftRows(0,1,2,3) ，而是用了一个闭包变量 shift_pattern 。
结合控制流和行为验证，可以确认变种为：
也就是说：
PyInit_runtime_codec @ 0x180009BE0
0x180007470
0x180007690
0x180002740
0x1800028E0
shift_pattern = (0, 1, 3, 2)


第 0 行左移 0
第 1 行左移 1
第 2 行左移 3
第 3 行左移 2
除此之外，其它结构仍然是标准 AES-128 风格：
S-box 是标准 AES S-box
MixColumns  是标准 AES
key schedule 是标准 AES-128
共 10 轮
最后一轮没有 MixColumns
所以这题本质上是“只改了一处 ShiftRows  的 AES-128”。
6. 拿到 target_ciphertext

verify_flag  会把加密结果与 target_ciphertext  比较，因此只要把这个目标密文拿到，就能离线逆。
这一步最简单的实战方法是：
在模块初始化阶段或 verify_flag  中对 PyBytes_FromStringAndSize  下断
看哪一块 48 字节缓冲区最终被当成目标密文使用
恢复出的目标密文为：
总长度是 48 字节，也就是 3 个 block，因此：
7. 逆变种 AES

现在我们已经有了全部关键材料：
AES key
target_ciphertext
变种点 ShiftRows = (0,1,3,2)
剩下的就是写逆脚本。
a5324678a39df1021d201826fef1b8e8
5c86030f5146aa2f8aad843758fd5c2d
1984c40892cb5c3db4194f091e46f099
target_block_count = 3


7.1 逆向思路

因为除了 ShiftRows  之外都和标准 AES 一样，所以解密流程也是标准 AES 逆轮，只需把 InvShiftRows  改成对应
这个 pattern。
逆轮流程：
1. AddRoundKey(round10)
2. InvShiftRows
3. InvSubBytes
4. 对 round 9 到 round 1：
AddRoundKey
InvMixColumns
InvShiftRows
InvSubBytes
5. AddRoundKey(round0)
最后把明文去掉 PKCS#7  padding 即可。
7.2 完整 exp

```python
from tools.variant_cipher import _SBOX, _expand_round_keys
AES_KEY = bytes.fromhex("1145141919810abc1145141919810cba")
TARGET_CT = bytes.fromhex(
    "a5324678a39df1021d201826fef1b8e8"
    "5c86030f5146aa2f8aad843758fd5c2d"
    "1984c40892cb5c3db4194f091e46f099"
)
SHIFT = (0, 1, 3, 2)
inv_sbox = [0] * 256
for index, value in enumerate(_SBOX):
    inv_sbox[value] = index
def gmul(a: int, b: int) -> int:
    result = 0
    for _ in range(8):
        if b & 1:
            result ^= a
        hi = a & 0x80
        a = (a << 1) & 0xFF
        if hi:
            a ^= 0x1B
        b >>= 1
    return result


def xor_round_key(state, round_key):
    return [(state[i] ^ round_key[i]) & 0xFF for i in range(16)]
def inv_sub_bytes(state):
    return [inv_sbox[x] for x in state]
def inv_shift_rows(state):
    src = state[:]
    out = state[:]
    for row, shift in enumerate(SHIFT):
        for col in range(4):
            out[col * 4 + row] = src[((col - shift) % 4) * 4 + row]
    return out
def inv_mix_columns(state):
    out = state[:]
    for col in range(4):
        base = col * 4
        a0, a1, a2, a3 = state[base:base + 4]
        out[base + 0] = gmul(a0, 14) ^ gmul(a1, 11) ^ gmul(a2, 13) ^ gmul(a3, 9)
        out[base + 1] = gmul(a0, 9) ^ gmul(a1, 14) ^ gmul(a2, 11) ^ gmul(a3, 13)
        out[base + 2] = gmul(a0, 13) ^ gmul(a1, 9) ^ gmul(a2, 14) ^ gmul(a3, 11)
        out[base + 3] = gmul(a0, 11) ^ gmul(a1, 13) ^ gmul(a2, 9) ^ gmul(a3, 14)
    return [x & 0xFF for x in out]
ROUND_KEYS = _expand_round_keys(AES_KEY)
def decrypt_block(block: bytes) -> bytes:
    state = list(block)
    state = xor_round_key(state, ROUND_KEYS[10])
    state = inv_shift_rows(state)
    state = inv_sub_bytes(state)
    for round_index in range(9, 0, -1):
        state = xor_round_key(state, ROUND_KEYS[round_index])
        state = inv_mix_columns(state)
        state = inv_shift_rows(state)
        state = inv_sub_bytes(state)
    state = xor_round_key(state, ROUND_KEYS[0])
    return bytes(state)
plaintext = b"".join(
    decrypt_block(TARGET_CT[offset:offset + 16])
    for offset in range(0, len(TARGET_CT), 16)
)


运行结果：
8. 最终答案

pad = plaintext[-1]
payload = plaintext[:-pad]
flag = f"SpiritCTF{{{payload.decode('ascii')}}}"
print(flag)
```

SpiritCTF{Sh1fT_r0w3_aRe_NoT_wHa7_Y0u_eXp3cT}
SpiritCTF{Sh1fT_r0w3_aRe_NoT_wHa7_Y0u_eXp3cT}

## 方法总结

- 核心技巧：先拆 loader 的运行时解密与嵌入式 Python 调用链，再分析释放出的 `.pyd` 中的 AES 变体。
- 识别信号：Windows 程序大量导入 `python312.dll` API、字符串中出现模块名和函数名时，应优先判断真正逻辑是否被转移到 Python 扩展模块。
- 复用要点：必须使用 loader 实际释放的扩展文件，避免旧构建文件与外层 key 或目标密文不匹配；AES 变体题应先确认轮函数中哪一步被改动，再写对应逆变换。
