# Findme

## 题目简述

外层程序展示的明文均为 fake flag，真正逻辑藏在一个被扩展为 32 位槽位数组的 PE 文件中。数组开头每隔 4 字节出现 `4d 00 00 00 5a 00 00 00 90 00 00 00`，取每个槽位的第一个字节后就是 `MZ 90` 文件头。

导出内层 PE 后仍有多组互补条件跳转花指令，清理后可恢复一个魔改 RC4。使用密钥 `deadbeef` 对给定 32 字节密文执行逆变换即可得到 flag。

## 解题过程

### 提取内层 PE

在 IDA 中定位可疑 buffer，确认其有效字节按 4 字节间隔存储。把该区域导出为 `raw.bin`，再抽取偏移 $0,4,8,\ldots$：

```python
raw = open("raw.bin", "rb").read()
embedded = bytes(raw[offset] for offset in range(0, len(raw), 4))

assert embedded.startswith(b"MZ")
open("dump.exe", "wb").write(embedded)
```

### 清理互补跳转花指令

内层 PE 的 `.text` 中重复出现：

```asm
jz  short target     ; 74 03
jnz short target     ; 75 01
db  0C7h
target:
```

无论零标志为何，控制流都会到同一个 `target`，中间的 `0xc7` 永远不会执行。可以把完整的 5 字节模式改成 NOP。为避免误伤正常的 `74 03`，脚本同时核对后续三个字节：

```python
import ida_bytes
import idautils
import idc

code_start = code_end = None
for segment in idautils.Segments():
    if idc.get_segm_name(segment) == ".text":
        code_start = idc.get_segm_start(segment)
        code_end = idc.get_segm_end(segment)
        break

assert code_start is not None
pattern = b"\x74\x03\x75\x01\xc7"

for address in range(code_start, code_end - len(pattern) + 1):
    if ida_bytes.get_bytes(address, len(pattern)) == pattern:
        ida_bytes.patch_bytes(address, b"\x90" * len(pattern))
```

修补后在真实函数头按 `P` 重新建立函数，再按 `F5` 反编译。算法与 RC4 类似，但有两处关键变化：

- S 盒初始化为 `s[i] = (256 - i) & 0xff`，而不是 `s[i] = i`；
- PRGA 对密文执行 `data[n] -= s[256 - t]`，而不是与 `s[t]` 异或。

### 还原明文

```python
key = b"deadbeef"
ciphertext = bytes([
    0x7D, 0x2B, 0x43, 0xA9, 0xB9, 0x6B, 0x93, 0x2D,
    0x9A, 0xD0, 0x48, 0xC8, 0xEB, 0x51, 0x59, 0xE9,
    0x74, 0x68, 0x8A, 0x45, 0x6B, 0xBA, 0xA7, 0x16,
    0xF1, 0x10, 0x74, 0xD5, 0x41, 0x3C, 0x67, 0x7D,
])

s = [(256 - i) & 0xFF for i in range(256)]
j = 0
for i in range(256):
    j = (j + s[i] + key[i % len(key)]) % 256
    s[i], s[j] = s[j], s[i]

i = j = 0
plaintext = bytearray()
for value in ciphertext:
    i = (i + 1) % 256
    j = (j + s[i]) % 256
    s[i], s[j] = s[j], s[i]
    t = (s[i] + s[j]) % 256
    assert t != 0  # 本题数据不会触发原 C 代码的 s[256] 越界。
    plaintext.append((value - s[256 - t]) & 0xFF)

print(plaintext.decode())
```

得到：

```text
hgame{Fl0w3rs_Ar3_Very_fr4grant}
```

## 方法总结

- fake flag 只是在干扰字符串搜索；看到 `MZ`、`PE` 等文件魔数应优先判断是否嵌入了第二阶段程序。
- “每 4 字节取 1 字节”来源于数据布局证据，而不是盲目删除所有零字节；后者可能误删内层文件本来就有的零。
- 批量去花应匹配完整、语义明确的指令序列，不能只凭一个短字节前缀全局替换。
- 识别为 RC4 家族后仍要逐项比较 S 盒初始化、KSA、PRGA、取流下标与组合运算，任何一处魔改都会改变输出。
