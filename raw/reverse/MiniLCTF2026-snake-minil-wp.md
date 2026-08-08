# snake_minil

## 题目简述

附件是贪吃蛇游戏程序。表面上只需控制蛇吃到食物，实际上每次“有效按键变化”都会对 `.rdata` 中的 `0x480` 字节隐藏 blob 执行一次变换。前 10 个食物位置固定，程序在吃到食物时计算 blob 的 MD5；只有路线对应的操作序列使其命中真实 MD5，才会用当前 blob 派生密钥并进入隐藏 finalizer。

题目还包含一个调试态诱饵 MD5，它会显示 `miniL{route_only}`，但正常无调试运行不会接受这条链。因此完整解法必须同时恢复按键状态机、blob 变换、key 派生和最终分组解密。

## 解题过程

### 还原按键到 blob 变换的映射

变换字母表为 `PALS`，四种操作定义如下：

```text
A: 每个字节模 256 加 0x1e
S: 每个字节模 256 减 0x66
L: 每个字节循环左移 3 位
P: 整个 blob 循环左移 6 字节
```

实际按键映射为：

```text
U -> P
D -> A
L -> L
R -> S
```

这里的易错点是，程序按 `lastChangedKey` 去重，而不是按蛇的 `moveDir` 去重。`lastChangedKey` 初始为 `None`，即使初始移动方向已经是 Right，开局第一次 `R` 仍会执行 `S`。连续相同按键不会再触发变换；与当前移动方向相反的键无效，也不会改变 blob。

### 根据固定食物恢复路线

蛇的初始坐标是 `(10,10)`，初始方向为 Right。前 10 个食物坐标为：

```text
(17,0), (12,10), (3,11), (15,9), (13,18),
(4,10), (12,10), (14,16), (3,7), (19,14)
```

对每段采用“先水平、再垂直”的最短路，得到 143 tick 按键：

```text
RRRRRRRUUUUUUUUUU
LLLLLDDDDDDDDDD
LLLLLLLLLD
RRRRRRRRRRRRUU
LLDDDDDDDDD
LLLLLLLLLUUUUUUUU
RRRRRRRR
RRDDDDDD
LLLLLLLLLLLUUUUUUUUU
RRRRRRRRRRRRRRRRDDDDDDD
```

压缩连续相同按键后，真正会触发 blob 变化的键序列为：

```text
RULDLDRULDLURDLURD
```

对应操作序列：

```text
SPLALASPLALPSALPSA
```

对初始 blob 依次执行后，真实目标 MD5 为：

```text
cac0dfcf4b795ee7436b17721d2411e1
```

等价验证代码：

```python
import hashlib

for op in "SPLALASPLALPSALPSA":
    if op == "A":
        blob = bytes((x + 0x1E) & 0xFF for x in blob)
    elif op == "S":
        blob = bytes((x - 0x66) & 0xFF for x in blob)
    elif op == "L":
        blob = bytes(((x << 3) & 0xFF) | (x >> 5) for x in blob)
    elif op == "P":
        blob = blob[6:] + blob[:6]

assert hashlib.md5(blob).hexdigest() == "cac0dfcf4b795ee7436b17721d2411e1"
```

调试态诱饵 MD5 `f7e0250119112e4c140ee5a21fa40025` 只对应真实操作前缀 `SPLA`，所以在第二个食物附近就可能出现假成功。这个输出不能代替完整路线的 MD5 验证。

### 从最终 blob 派生 128-bit key

MD5 命中后，程序从当前 blob 逐字节混合四个 32-bit lane，然后再异或 blob 在 `0x40` 处的 16 字节：

```python
import struct

MASK = 0xFFFFFFFF


def rol32(value, amount):
    return ((value << amount) & MASK) | (value >> (32 - amount))


key = [0x243F6A88, 0x85A308D3, 0x13198A2E, 0x03707344]
for i, byte in enumerate(blob):
    lane = i & 3
    mix = (
        byte + 0x9E3779B9
        + ((key[(lane + 1) & 3] << 6) & MASK)
        + (key[(lane + 3) & 3] >> 2)
    ) & MASK
    key[lane] = rol32(key[lane] ^ mix, ((i + lane) & 7) + 5)

for i in range(4):
    key[i] ^= struct.unpack_from("<I", blob, 0x40 + 4 * i)[0]
```

正确 blob 得到：

```text
032a05e4 866a4de5 a815d7ef 1a8c3ff3
```

### 逆转 finalizer

最终 16 字节密文为：

```text
a4a0bcf4021dbd97e715f4cb73902a68
```

先逆转四个 32-bit word 之间的 XOR 后处理，再对后一对 word 做 32 轮、前一对做 32 轮 XTEA-like 逆运算：

```python
import struct

MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9
KEY = [0x032A05E4, 0x866A4DE5, 0xA815D7EF, 0x1A8C3FF3]
target = bytes.fromhex("a4a0bcf4021dbd97e715f4cb73902a68")


def mix(x, total, key_word):
    return (((((x << 4) & MASK) ^ (x >> 5)) + x) & MASK) ^ (
        (total + key_word) & MASK
    )


f0, f1, f2, f3 = struct.unpack("<4I", target)
v2 = f1 ^ f2
v0 = f0 ^ v2
v3 = f3 ^ f0
v1 = f1 ^ v3

total = DELTA * 64 & MASK
for _ in range(32):
    v3 = (v3 - mix(v2, total, KEY[(total >> 11) & 3])) & MASK
    total = (total - DELTA) & MASK
    v2 = (v2 - mix(v3, total, KEY[total & 3])) & MASK

total = DELTA * 32 & MASK
for _ in range(32):
    v1 = (v1 - mix(v0, total, KEY[(total >> 11) & 3])) & MASK
    total = (total - DELTA) & MASK
    v0 = (v0 - mix(v1, total, KEY[total & 3])) & MASK

print(struct.pack("<4I", v0, v1, v2, v3))
```

输出：

```text
b'miniL{r0ut3_Snk}'
```

## 方法总结

- 核心技巧：将游戏路线压缩成“方向变化”序列，用它重放隐藏 blob 变换，通过 MD5 闭环验证后再还原 key 派生和 finalizer。
- 识别信号：游戏按键与 `.rdata` 大 blob 有稳定变换关系，且吃食物时计算 blob 哈希，说明路径不是单纯通关条件，而是加密算法的输入。
- 复用要点：必须区分当前移动方向与最后有效按键；调试器特有的提前成功分支应被当成 decoy，最终结果应同时通过真实 MD5、派生 key 和原 finalizer 验证。
