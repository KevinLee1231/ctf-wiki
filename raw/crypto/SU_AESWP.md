# SU_AES

## 题目简述
附件是变种 AES 服务。`chal.py` 允许重置 seed/key，`AES.py` 中 `change(seed, key)` 会在只给 seed 时通过 `Random(seed).choices(self.Sbox, k=len(self.Sbox))` 重采样当前 S-box，却保留旧 round keys，导致“当前 S-box”和“生成轮密钥时的 S-box”脱钩。利用这个状态不一致可以把 S-box 压成常值，恢复最后一轮密钥并进一步恢复初始置换；密文只是验证材料，重点是服务源码中的状态更新逻辑。

## 解题过程
题目的真正漏洞点

这题表面上是“你可以改 S-box”，但致命点其实不是改没改，而是改的时候旧轮密钥还留着。

AES.change(s, k) 的行为分成两半：

if s:

$$
self.Sbox = Random(s).choices(self.Sbox, k=len(self.Sbox))
$$

if k:

self.change_key(k)

也就是说：

- 只给 seed 时，会把当前 S-box 重新采样一遍；

- 但如果不给 key ，旧的 round keys 完全不动。

这就把“当前加密用的 S-box”和“当年生成轮密钥时用的 S-box”人为拆开了。

先把这个 change(seed) 看成一个函数

设当前 S-box 是一个列表 T ，长度 256。 对固定 seed 来说，Random(seed).choices(...)
实际上等价于固定出一个索引函数：

f_seed : {0..255} -> {0..255}

一次 change(seed) 之后，新 S-box 就是：

$$
T'(x) = T(f_seed(x))
$$

如果连续用同一个 seed 多次，那么就会变成：

$$
T_t(x) = T(f_seed^t(x))
$$

因此当前 S-box 的值域是：

$$
Im(T_t) = T(Im(f_seed^t))
$$

右边这个 Im(f_seed^t) 完全可以离线算出来，因为它只和 Python 的 Random(seed) 有关，
和题目的 secret 无关。

第一阶段：先拿最后一轮轮密钥 K10

最后一轮 AES 没有 MixColumns ，所以它的结构非常干净：

$$
C = ShiftRows(SubBytes(S9)) xor K10
$$

如果我们能把当前 S-box 压成常值 u ，那么：

$$
SubBytes(*) = u
$$

$$
ShiftRows([u]*16) = [u]*16
$$

于是任意明文都会得到：

$$
C = [u]*16 xor K10
$$

这时候 K10 = C xor [u]*16 ，问题只剩下这个常值 u 是多少。

怎么把 S-box 压成常值

离线搜索 seed，使得对应的函数图只有一个吸收点。 我这里找到的参数是：
- collapse seed: 138188
- 连续调用次数：18

它的 f^18 的像集大小正好收缩到 1。

怎么知道常值 u

这里不去猜原始密钥，直接重建一个“已知密钥版本”的常值 AES：

1. 先把 S-box 压成常值；

2. 再调用一次菜单 1，但是这次只传 key=1 ，让它在“常值 S-box”下重排轮密钥；

3. 由于 master key 已知，常值只可能是 0..255 中某一个，直接本地枚举 256 种常值即可。

拿到 u 之后，reset 回原始状态，再压一次常值，查一次加密，就能恢复：

$$
K10 = C xor [u]*16
$$

第二阶段：恢复最初那张打乱后的 S-box

有了 K10 ，我们就能把任意密文的最后一轮 key 拆掉：

$$
InvShiftRows(C xor K10) = T(S9)
$$

右边每个字节一定落在当前 S-box 的值域里，所以如果在某个固定的当前 S-box 下发很多随机明文，
收集

InvShiftRows(C xor K10)

里出现过的所有字节，得到的就是：

Im(T)

而上一节已经说过，当前值域满足：

$$
Im(T) = P(Im(f_seed^t))
$$

这里 P 表示最初那张未知的打乱 S-box，它本身是一个 256 位置上的置换。

变成一个“指纹匹配”问题

我们挑若干个 probe seed。对每个 probe：

- 本地先算出索引集合 I = Im(f_seed) ；

- 远程恢复出值集合 V = P(I) 。

这样对于任意索引 x ，都可以写出一个布尔指纹：

$$
sig_idx(x) = [x in I_1, x in I_2, ..., x in I_n]
$$

对于任意字节值 y ，也有对应的值指纹：

$$
sig_val(y) = [y in V_1, y in V_2, ..., y in V_n]
$$

因为 V_i = P(I_i) ，所以一定有：

$$
sig_val(P(x)) = sig_idx(x)
$$

只要 probe 选得好，使得 256 个位置的指纹两两不同，就能直接一一配对，把整个 P 重建出来。

我这里离线挑出的 probe seeds 是：

[1052, 3745, 4616, 446, 1695, 1325, 4261, 1897, 891, 4770, 1414, 2]

这 12 组一次就够，且全部用 t=1 ，实现上最省事。

第三阶段：从 K10 反推主密钥

拿到完整的 P 以后，AES-128 的 key schedule 就变回了普通可逆过程。

因为最后一轮轮密钥已经知道，所以按 key expansion 逆着推回去即可：
- w[43] -> w[0]

- 遇到 i % 4 == 0 时，用当前恢复出的自定义 S-box P

- 最终得到 16 字节主密钥

这一步完全不需要再和远程交互。

最后解 flag

有了：

- 完整 S-box P

- 全部 round keys

就可以本地实现逆过程：

AddRoundKey

InvShiftRows

InvSubBytes

...

InvMixColumns

把题目一开始给出的 flag 密文逐块解开，再做 PKCS#7 去填充即可。

实战时踩到的一个小坑

本地 chal.py 这里写的是：

$$
k = int(input('[x] your key: ') or 0, 16) or None
$$

如果真的发空行，int(0, 16) 会直接报 TypeError 。 所以脚本里不要发送空串，而是统一发
字符串 "0" ，这样解析出来仍然是 None ，本地和远程都更稳。

AES.py

```python
from random import Random

# learnt from http://cs.ucsb.edu/~koc/cs178/projects/JT/aes.c
xtime = lambda a: (((a << 1) ^ 0x1B) & 0xFF) if (a & 0x80) else (a << 1)

Rcon = (
0x00, 0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40,
0x80, 0x1B, 0x36, 0x6C, 0xD8, 0xAB, 0x4D, 0x9A,
0x2F, 0x5E, 0xBC, 0x63, 0xC6, 0x97, 0x35, 0x6A,
0xD4, 0xB3, 0x7D, 0xFA, 0xEF, 0xC5, 0x91, 0x39,
)

def text2matrix(text):
matrix = [[] for _ in range(4)]
for i in range(16):
byte = (text >> (8 * (15 - i))) & 0xFF
matrix[i % 4].append(byte)
return matrix

def matrix2text(matrix):
text = 0
for i in range(4):
for j in range(4):
text |= (matrix[j][i] << (120 - 8 * (4 * i + j)))
return text

class AES:
def __init__(self, master_key, seed=None):
self.Sbox = [0x63, 0x7C, 0x77, 0x7B, 0xF2, 0x6B, 0x6F, 0xC5, 0x30,
0x01, 0x67, 0x2B, 0xFE, 0xD7, 0xAB, 0x76,
0xCA, 0x82, 0xC9, 0x7D, 0xFA, 0x59, 0x47, 0xF0, 0xAD, 0xD4, 0xA2,
0xAF, 0x9C, 0xA4, 0x72, 0xC0,
0xB7, 0xFD, 0x93, 0x26, 0x36, 0x3F, 0xF7, 0xCC, 0x34, 0xA5, 0xE5,
0xF1, 0x71, 0xD8, 0x31, 0x15,

0x04, 0xC7, 0x23, 0xC3, 0x18, 0x96, 0x05, 0x9A, 0x07, 0x12, 0x80,
0xE2, 0xEB, 0x27, 0xB2, 0x75,
0x09, 0x83, 0x2C, 0x1A, 0x1B, 0x6E, 0x5A, 0xA0, 0x52, 0x3B, 0xD6,
0xB3, 0x29, 0xE3, 0x2F, 0x84,
0x53, 0xD1, 0x00, 0xED, 0x20, 0xFC, 0xB1, 0x5B, 0x6A, 0xCB, 0xBE,
0x39, 0x4A, 0x4C, 0x58, 0xCF,
0xD0, 0xEF, 0xAA, 0xFB, 0x43, 0x4D, 0x33, 0x85, 0x45, 0xF9, 0x02,
0x7F, 0x50, 0x3C, 0x9F, 0xA8,
0x51, 0xA3, 0x40, 0x8F, 0x92, 0x9D, 0x38, 0xF5, 0xBC, 0xB6, 0xDA,
0x21, 0x10, 0xFF, 0xF3, 0xD2,
0xCD, 0x0C, 0x13, 0xEC, 0x5F, 0x97, 0x44, 0x17, 0xC4, 0xA7, 0x7E,
0x3D, 0x64, 0x5D, 0x19, 0x73,
0x60, 0x81, 0x4F, 0xDC, 0x22, 0x2A, 0x90, 0x88, 0x46, 0xEE, 0xB8,
0x14, 0xDE, 0x5E, 0x0B, 0xDB,
0xE0, 0x32, 0x3A, 0x0A, 0x49, 0x06, 0x24, 0x5C, 0xC2, 0xD3, 0xAC,
0x62, 0x91, 0x95, 0xE4, 0x79,
0xE7, 0xC8, 0x37, 0x6D, 0x8D, 0xD5, 0x4E, 0xA9, 0x6C, 0x56, 0xF4,
0xEA, 0x65, 0x7A, 0xAE, 0x08,
0xBA, 0x78, 0x25, 0x2E, 0x1C, 0xA6, 0xB4, 0xC6, 0xE8, 0xDD, 0x74,
0x1F, 0x4B, 0xBD, 0x8B, 0x8A,
0x70, 0x3E, 0xB5, 0x66, 0x48, 0x03, 0xF6, 0x0E, 0x61, 0x35, 0x57,
0xB9, 0x86, 0xC1, 0x1D, 0x9E,
0xE1, 0xF8, 0x98, 0x11, 0x69, 0xD9, 0x8E, 0x94, 0x9B, 0x1E, 0x87,
0xE9, 0xCE, 0x55, 0x28, 0xDF,
0x8C, 0xA1, 0x89, 0x0D, 0xBF, 0xE6, 0x42, 0x68, 0x41, 0x99, 0x2D,
0x0F, 0xB0, 0x54, 0xBB, 0x16]

Random(seed).shuffle(self.Sbox)

self.change_key(master_key)

def change_key(self, master_key):
self.round_keys = text2matrix(master_key)
for i in range(4, 4 * 11):
self.round_keys.append([])
if i % 4 == 0:
temp = [
self.round_keys[i - 4][0] ^ self.Sbox[self.round_keys[i -
1][1]] ^ Rcon[i // 4],
self.round_keys[i - 4][1] ^ self.Sbox[self.round_keys[i -
1][2]],
self.round_keys[i - 4][2] ^ self.Sbox[self.round_keys[i -
1][3]],
self.round_keys[i - 4][3] ^ self.Sbox[self.round_keys[i -
1][0]],
]
else:

temp = [
self.round_keys[i - 4][j] ^ self.round_keys[i - 1][j] for
j in range(4)
]
self.round_keys[i] = temp

def encrypt(self, plaintext):
state = text2matrix(plaintext)
self.add_round_key(state, self.round_keys[:4])
for i in range(1, 10):
self.sub_bytes(state)
self.shift_rows(state)
self.mix_columns(state)
self.add_round_key(state, self.round_keys[4*i:4*(i+1)])
self.sub_bytes(state)
self.shift_rows(state)
self.add_round_key(state, self.round_keys[40:])
return matrix2text(state)

def add_round_key(self, s, k):
for i in range(4):
for j in range(4):
s[i][j] ^= k[i][j]

def sub_bytes(self, s):
for i in range(4):
for j in range(4):
s[i][j] = self.Sbox[s[i][j]]

def shift_rows(self, s):
s[0][1], s[1][1], s[2][1], s[3][1] = s[1][1], s[2][1], s[3][1], s[0][1]
s[0][2], s[1][2], s[2][2], s[3][2] = s[2][2], s[3][2], s[0][2], s[1][2]
s[0][3], s[1][3], s[2][3], s[3][3] = s[3][3], s[0][3], s[1][3], s[2][3]

def mix_columns(self, s):
for i in range(4):
t = s[i][0] ^ s[i][1] ^ s[i][2] ^ s[i][3]
u = s[i][0]
s[i][0] ^= t ^ xtime(s[i][0] ^ s[i][1])
s[i][1] ^= t ^ xtime(s[i][1] ^ s[i][2])
s[i][2] ^= t ^ xtime(s[i][2] ^ s[i][3])
s[i][3] ^= t ^ xtime(s[i][3] ^ u)

def encrypt_ecb(self, data: bytes) -> bytes:
if not isinstance(data, (bytes, bytearray)):
raise TypeError("data must be bytes-like")
if len(data) % 16 != 0:

raise ValueError("data length must be multiple of 16 when
pad=False")
out = b''
for i in range(0, len(data), 16):
out += self.encrypt(int.from_bytes(data[i : i + 16])).to_bytes(16)
return out

def change(self, s=None, k=None):
if s:
self.Sbox = Random(s).choices(self.Sbox, k=len(self.Sbox))
if k:
self.change_key(k)
```

solve.py

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import os
import re
from dataclasses import dataclass
from random import Random

from Crypto.Util.Padding import unpad
from pwn import context, process, remote

from AES import AES, Rcon, matrix2text, text2matrix

context.log_level = "error"

COLLAPSE_SEED = 138188
COLLAPSE_STEPS = 18
PROBE_SEEDS = [1052, 3745, 4616, 446, 1695, 1325, 4261, 1897, 891, 4770, 1414,
2]
SAMPLE_BLOCKS = 1024

def xor_bytes(a: bytes, b: bytes) -> bytes:
return bytes(x ^ y for x, y in zip(a, b))

def inv_shift_rows_state(state: list[list[int]]) -> None:
state[0][1], state[1][1], state[2][1], state[3][1] = (
state[3][1],
state[0][1],
state[1][1],
state[2][1],
)
state[0][2], state[1][2], state[2][2], state[3][2] = (
state[2][2],
state[3][2],
state[0][2],
state[1][2],
)
state[0][3], state[1][3], state[2][3], state[3][3] = (
state[1][3],
state[2][3],
state[3][3],
state[0][3],
)

def inv_shift_rows_block(block: bytes) -> bytes:
state = text2matrix(int.from_bytes(block, "big"))
inv_shift_rows_state(state)
return matrix2text(state).to_bytes(16, "big")

def gf_mul(a: int, b: int) -> int:
out = 0
for _ in range(8):
if b & 1:
out ^= a
high = a & 0x80
a = (a << 1) & 0xFF
if high:
a ^= 0x1B
b >>= 1
return out

def inv_mix_columns_state(state: list[list[int]]) -> None:
for i in range(4):
a0, a1, a2, a3 = state[i]
state[i][0] = gf_mul(a0, 14) ^ gf_mul(a1, 11) ^ gf_mul(a2, 13) ^
gf_mul(a3, 9)
state[i][1] = gf_mul(a0, 9) ^ gf_mul(a1, 14) ^ gf_mul(a2, 11) ^
gf_mul(a3, 13)

state[i][2] = gf_mul(a0, 13) ^ gf_mul(a1, 9) ^ gf_mul(a2, 14) ^
gf_mul(a3, 11)
state[i][3] = gf_mul(a0, 11) ^ gf_mul(a1, 13) ^ gf_mul(a2, 9) ^
gf_mul(a3, 14)

def add_round_key(state: list[list[int]], round_key: list[list[int]]) -> None:
for i in range(4):
for j in range(4):
state[i][j] ^= round_key[i][j]

def invert_key_schedule(last_round_key: bytes, sbox: list[int]) ->
tuple[bytes, list[list[int]]]:
words = [[0] * 4 for _ in range(44)]
tail = text2matrix(int.from_bytes(last_round_key, "big"))
for i in range(4):
words[40 + i] = tail[i][:]

for i in range(43, 3, -1):
if i % 4 == 0:
g = [
sbox[words[i - 1][1]] ^ Rcon[i // 4],
sbox[words[i - 1][2]],
sbox[words[i - 1][3]],
sbox[words[i - 1][0]],
]
words[i - 4] = [words[i][j] ^ g[j] for j in range(4)]
else:
words[i - 4] = [words[i][j] ^ words[i - 1][j] for j in range(4)]

master_key = matrix2text(words[:4]).to_bytes(16, "big")
return master_key, words

def decrypt_block(block: bytes, round_keys: list[list[int]], sbox: list[int]) -
> bytes:
inv_sbox = [0] * 256
for i, value in enumerate(sbox):
inv_sbox[value] = i

state = text2matrix(int.from_bytes(block, "big"))
add_round_key(state, round_keys[40:44])
inv_shift_rows_state(state)
for i in range(4):
for j in range(4):
state[i][j] = inv_sbox[state[i][j]]

for round_id in range(9, 0, -1):
add_round_key(state, round_keys[4 * round_id : 4 * (round_id + 1)])
inv_mix_columns_state(state)
inv_shift_rows_state(state)
for i in range(4):
for j in range(4):
state[i][j] = inv_sbox[state[i][j]]

add_round_key(state, round_keys[:4])
return matrix2text(state).to_bytes(16, "big")

def decrypt_ecb(ciphertext: bytes, round_keys: list[list[int]], sbox:
list[int]) -> bytes:
blocks = []
for i in range(0, len(ciphertext), 16):
blocks.append(decrypt_block(ciphertext[i : i + 16], round_keys, sbox))
return b"".join(blocks)

def probe_set(seed: int) -> set[int]:
return set(Random(seed).choices(range(256), k=256))

PROBE_INDEX_SETS = {seed: probe_set(seed) for seed in PROBE_SEEDS}
PROBE_SIGNATURE_TO_INDEX = {}
for idx in range(256):
sig = tuple(idx in PROBE_INDEX_SETS[seed] for seed in PROBE_SEEDS)
if sig in PROBE_SIGNATURE_TO_INDEX:
raise RuntimeError("probe signatures are not unique")
PROBE_SIGNATURE_TO_INDEX[sig] = idx

@dataclass
class SolveResult:
flag_ciphertext: bytes
k10: bytes
sbox: list[int]
master_key: bytes
plaintext: bytes

class Oracle:
def __init__(self, io):
self.io = io
self.flag_ciphertext = self._read_flag_ciphertext()

def _read_flag_ciphertext(self) -> bytes:
data = self.io.recvuntil(b"[x] >", drop=False)
match = re.search(rb"flag ciphertext \(in hex\): ([0-9a-f]+)", data)
if not match:
raise RuntimeError("failed to read initial flag ciphertext")
return bytes.fromhex(match.group(1).decode())

def change(self, seed: int | None = None, key: int | None = None) -> None:
self.io.sendline(b"1")
self.io.recvuntil(b"your seed: ")
self.io.sendline(b"0" if seed is None else format(seed, "x").encode())
self.io.recvuntil(b"your key: ")
self.io.sendline(b"0" if key is None else format(key, "x").encode())
self.io.recvuntil(b"[x] >")

def encrypt(self, msg: bytes) -> bytes:
self.io.sendline(b"2")
self.io.recvuntil(b"your message: ")
self.io.sendline(msg.hex().encode())
data = self.io.recvuntil(b"[x] >", drop=False)
match = re.search(rb"ciphertext \(in hex\): ([0-9a-f]+)", data)
if not match:
raise RuntimeError("failed to read ciphertext")
return bytes.fromhex(match.group(1).decode())

def reset(self) -> None:
self.io.sendline(b"3")
self.io.recvuntil(b"[x] >")

def close(self) -> None:
try:
self.io.close()
except Exception:
pass

def find_constant_value(oracle: Oracle) -> tuple[int, bytes]:
for _ in range(COLLAPSE_STEPS):
oracle.change(seed=COLLAPSE_SEED)
oracle.change(key=1)
known_ct = oracle.encrypt(b"")[:16]

value = None
for candidate in range(256):
aes = AES(0)
aes.Sbox = [candidate] * 256

aes.change_key(1)
if aes.encrypt_ecb(bytes.fromhex("10101010101010101010101010101010"))
[:16] == known_ct:
value = candidate
break
if value is None:
raise RuntimeError("failed to identify constant S-box value")

oracle.reset()
for _ in range(COLLAPSE_STEPS):
oracle.change(seed=COLLAPSE_SEED)
original_ct = oracle.encrypt(b"")[:16]
k10 = xor_bytes(original_ct, bytes([value]) * 16)
return value, k10

def recover_image_set(oracle: Oracle, seed: int, k10: bytes) -> set[int]:
target_size = len(PROBE_INDEX_SETS[seed])
seen = set()

oracle.reset()
oracle.change(seed=seed)
while len(seen) < target_size:
plaintext = os.urandom(16 * SAMPLE_BLOCKS)
ciphertext = oracle.encrypt(plaintext)
for i in range(0, len(ciphertext), 16):
transformed = inv_shift_rows_block(xor_bytes(ciphertext[i : i +
16], k10))
seen.update(transformed)
return seen

def recover_sbox(oracle: Oracle, k10: bytes) -> list[int]:
value_sets = {seed: recover_image_set(oracle, seed, k10) for seed in
PROBE_SEEDS}
sbox = [0] * 256
for value in range(256):
sig = tuple(value in value_sets[seed] for seed in PROBE_SEEDS)
sbox[PROBE_SIGNATURE_TO_INDEX[sig]] = value
return sbox

def verify_recovered_state(oracle: Oracle, master_key: bytes, sbox: list[int])
-> None:
oracle.reset()
plaintext = os.urandom(64)
server_ct = oracle.encrypt(plaintext)

aes = AES(0)
aes.Sbox = sbox[:]
aes.change_key(int.from_bytes(master_key, "big"))
local_ct = aes.encrypt_ecb(plaintext + bytes([16]) * 16)
if server_ct != local_ct:
raise RuntimeError("verification failed: recovered state does not
match oracle")

def solve(oracle: Oracle, verify: bool = True) -> SolveResult:
_, k10 = find_constant_value(oracle)
sbox = recover_sbox(oracle, k10)
master_key, round_keys = invert_key_schedule(k10, sbox)
if verify:
verify_recovered_state(oracle, master_key, sbox)
plaintext = unpad(decrypt_ecb(oracle.flag_ciphertext, round_keys, sbox),
16)
return SolveResult(oracle.flag_ciphertext, k10, sbox, master_key,
plaintext)

def build_io(args):
if args.local:
return process(["python3", "chal.py"],
cwd=os.path.dirname(os.path.abspath(__file__)))
return remote(args.host, args.port)

def parse_args() -> argparse.Namespace:
parser = argparse.ArgumentParser(description="Solve the shuffled-S-box AES
challenge")
parser.add_argument("--local", action="store_true", help="run against
local chal.py")
parser.add_argument("--host", default="<target>")
parser.add_argument("--port", type=int, default=10002)
parser.add_argument("--no-verify", action="store_true", help="skip final
oracle verification step")
return parser.parse_args()

def main() -> None:
args = parse_args()
io = build_io(args)
oracle = Oracle(io)
try:
result = solve(oracle, verify=not args.no_verify)
print(f"flag ciphertext = {result.flag_ciphertext.hex()}")

print(f"k10 = {result.k10.hex()}")
print(f"master key = {result.master_key.hex()}")
try:
print(f"flag = {result.plaintext.decode()}")
except UnicodeDecodeError:
print(f"flag bytes = {result.plaintext!r}")
finally:
oracle.close()

if __name__ == "__main__":
main()
```

## 方法总结
- 核心技巧：变种 AES 状态不一致攻击
- 识别信号：服务允许分别控制 seed/key，S-box 更新和 round key 更新不同步。
- 复用要点：先构造使 S-box 收缩到常值的 seed 序列，恢复 `K10`；再用多个 probe seed 做值域指纹匹配恢复置换。
