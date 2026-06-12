# SU_old_bin

## 题目简述
题目是 old firmware 固件逆向，flag 格式为 `flag{****}`。外层 `old.bin` 具有大量 `0x7f` 字节，说明整体先被固定字节 XOR；解开后得到自定义 `IMG0` 容器，容器里包含多个压缩段，其中主逻辑段是一个头部和 program header 被破坏的 ELF，需要修复 magic、LOAD 段偏移和 TLS 相关布局后才能正常分析。

修复后的主程序实现一个监听 `5534` 端口的网络服务：服务端生成 32 字节 challenge，经本地变换后发给客户端，再接收最多 64 字节输入并进入自定义块加密校验。解题目标是还原容器提取、ELF 修复和网络校验算法，反推出能通过校验的输入。

## 解题过程
从文件中发现有非常多的0x7F可以断定该固件被xor了0x7F

使用以下脚本去解密

```python
from pathlib import Path
src = Path('old.bin').read_bytes()
out = bytes(b ^ 0x7f for b in src)
Path('old_xor.bin').write_bytes(out)
```

解密后的文件是自定义 `IMG0` 容器，内部段偏移可直接按固定表提取：

该容器中有三个文件，使用如下脚本提取出来

```python
from pathlib import Path
p = Path('old_xor.bin').read_bytes()
segs = [
(0x2028, 0x4eeac, 'seg1.bin'),
(0x50ed4, 0x0bd0, 'seg2.bin'),
(0x51aa4, 0x1408, 'seg3.bin'),
]
for off, size, name in segs:
Path(name).write_bytes(p[off:off+size])
print(name, hex(off), hex(size))
```

去分析一下这个提取出来的固件可以发现这个一个xz的压缩文件

一共三个压缩文件但是其中两个文件大小非常小不是主要的逻辑，主要的逻辑在于seg1，解压缩后发
现是一个ELF文件但是文件头被魔改了，自己修复一下即可,继续分析发现第二个 LOAD 段的 p_offset
被故意错开了 `0x10000`，导致 TLS 段也跟着错了。修复方式就是把对应 program header 的 `p_offset` 加回 `0x10000`：

```python
from pathlib import Path
import struct

p = bytearray(Path('unpack/seg1_fixedmagic.elf').read_bytes())

phoff = struct.unpack_from('<Q', p, 0x20)[0]
phentsz = struct.unpack_from('<H', p, 0x36)[0]

for idx in [2, 5]:
off = phoff + phentsz * idx
val = struct.unpack_from('<Q', p, off + 8)[0]
struct.pack_into('<Q', p, off + 8, val + 0x10000)

Path('unpack/seg1_fixed_all.elf').write_bytes(p)
```

Main 函数首先创建 socket 并监听 `5534` 端口：

```c
fd = socket(AF_INET, SOCK_STREAM, 0);
bind(fd, "0.0.0.0:5534");
listen(fd, ...);
client = accept(fd, ...);
```

服务端使用本地 32 字节作为 challenge，处理后发送给客户端；随后接收客户端最多 64 字节响应，再调用校验函数：

```text
challenge[32] -> local_transform -> send(client)
recv(client, user_input, <=64)
validate(user_input, challenge, constants)
```

加密输入并且填充成64字节

进行三次 XOR 混合后，取出 16 字节转成 4 个 `uint32_t`；同时把 64 字节中间结果按 4 字节转成 `uint32_t` 数组：

继续加密并且取出16字节的数据转为4个int类型

将64字节的加密后的结果也没四字节转为Int类型

随后把两个 `uint32_t` 数组送入 block 变换，核心结构类似“常量表 + 自定义 S-box + 轮函数”的可逆块加密：

```text
state = xor_mix(padded_input, challenge)
words = bytes_to_u32(state)
for round in range(32):
    words = block_round(words, fk, ck[round], custom_sbox)
encrypted = u32_to_bytes(words)
```

写回加密后的结果并且进行Flag的校验

### Exp：

```python
from __future__ import annotations

import argparse
from dataclasses import dataclass
from itertools import product
from pathlib import Path
from typing import Dict, Iterable, List, Sequence

MASK64 = (1 << 64) - 1
MASK32 = (1 << 32) - 1

# Offsets inside the already-fixed ELF.
AES_SBOX_OFF = 0x7E6C0
TARGET_OFF = 0x7E7C0
KEY_OFF = 0x7E920
FK_OFF = 0x7E950
CK_OFF = 0x7E970
CUSTOM_SBOX_OFF = 0x7EA70

# Constants reconstructed from init_ctx / helper functions.
SEED_WORDS = [
0xFFF55731369D7563,
0x16E58EB22FBD5C72,
0x3632ED844C43F5B0,
0x390980A442221584,
]
SEED_MIX_INIT = 0x1234567890ABCDEF
SEED_FALLBACK = 0xDEADBEEFCAFEBABE

DEFAULT_ALLOWED = "abcdefghijklmnopqrstuvwxyz0123456789{}_"

@dataclass
class Constants:
aes_sbox: List[int]
aes_inv: List[int]
target: bytes
key_bytes: bytes
fk: List[int]
ck: List[int]
custom_sbox: List[int]

@dataclass
class Context:

state: List[int] # final mutated xoroshiro/xoroshiro-like state used by
validate()
tbl20: List[int] # 64 bytes
tbl28: List[int] # 64-byte permutation
tbl30: List[int] # 48 bytes

def rotl64(x: int, k: int) -> int:
x &= MASK64
return ((x << k) & MASK64) | (x >> (64 - k))

def rotl32(x: int, k: int) -> int:
x &= MASK32
return ((x << k) & MASK32) | (x >> (32 - k))

def rol8(x: int, k: int) -> int:
return ((x << k) & 0xFF) | (x >> (8 - k))

def splitmix64_next(box: List[int]) -> int:
box[0] = (box[0] + 0x9E3779B97F4A7C15) & MASK64
z = box[0]
z = ((z ^ (z >> 30)) * 0xBF58476D1CE4E5B9) & MASK64
z = ((z ^ (z >> 27)) * 0x94D049BB133111EB) & MASK64
z ^= z >> 31
return z & MASK64

def prng_next(state: Sequence[int]) -> tuple[int, List[int]]:
"""xoroshiro256** style next() used by challenge() and validate()."""
s0, s1, s2, s3 = state
result = rotl64((s1 * 5) & MASK64, 7)
result = (result * 9) & MASK64
t = (s1 << 17) & MASK64

s2 ^= s0
s3 ^= s1
s1 ^= s2
s0 ^= s3
s2 ^= t
s3 = rotl64(s3, 45)

return result, [s0 & MASK64, s1 & MASK64, s2 & MASK64, s3 & MASK64]

def load_constants(elf_path: Path) -> Constants:
data = elf_path.read_bytes()

aes_sbox = list(data[AES_SBOX_OFF:AES_SBOX_OFF + 256])
if len(aes_sbox) != 256:
raise ValueError("failed to read AES S-box")
aes_inv = [0] * 256
for i, b in enumerate(aes_sbox):
aes_inv[b] = i

target = data[TARGET_OFF:TARGET_OFF + 64]
key_bytes = data[KEY_OFF:KEY_OFF + 16]
fk = [int.from_bytes(data[FK_OFF + i * 8:FK_OFF + i * 8 + 8], "little") for
i in range(4)]
ck = [int.from_bytes(data[CK_OFF + i * 8:CK_OFF + i * 8 + 8], "little") for
i in range(32)]
custom_sbox = list(data[CUSTOM_SBOX_OFF:CUSTOM_SBOX_OFF + 256])

return Constants(
aes_sbox=aes_sbox,
aes_inv=aes_inv,
target=target,
key_bytes=key_bytes,
fk=fk,
ck=ck,
custom_sbox=custom_sbox,
)

def init_ctx(consts: Constants) -> Context:
"""Exact deterministic init_ctx() reconstruction."""
mixer = [SEED_MIX_INIT]
initial_state: List[int] = []

for seed in SEED_WORDS:
mixer[0] ^= (seed + 0x9E3779B97F4A7C15) & MASK64
initial_state.append(splitmix64_next(mixer))

if all(x == 0 for x in initial_state):
initial_state[0] = SEED_FALLBACK

# The real init function mutates ctx->state while generating the tables.
state = initial_state[:]
tbl20 = [0] * 64
tbl28 = [0] * 64
tbl30 = [0] * 48

for i in range(64):
tbl28[i] = i
r, state = prng_next(state)
tbl20[i] = ((r & 0xFF) ^ ((r >> 11) & 0xFF) ^ ((i - 0x5B) & 0xFF)) &
0xFF

# Fisher-Yates style shuffle from the end down to 1.
for i in range(63, 0, -1):
r, state = prng_next(state)
j = r % (i + 1)
tbl28[i], tbl28[j] = tbl28[j], tbl28[i]

for i in range(48):
r, state = prng_next(state)
t = ((r & 0xFF) ^ ((r >> 23) & 0xFF) ^ ((((7 * i) & 0xFF) + 0x3D) &
0xFF)) & 0xFF
t = (t + tbl20[i & 0x3F]) & 0xFF
t = consts.aes_sbox[t]
r2, state = prng_next(state)
t ^= r2 & 0xFF
# The binary uses a 64-bit rotate-left on a low-byte value and then
```

truncates.

```python
t = rotl64(t, (i % 7) + 1) & 0xFF
tbl30[i] = t

return Context(state=state, tbl20=tbl20, tbl28=tbl28, tbl30=tbl30)

def sbox_custom_byte(b: int, consts: Constants) -> int:
return consts.custom_sbox[(b + 0x37) & 0xFF]

def tau(word: int, consts: Constants) -> int:
word &= MASK32
return (
(sbox_custom_byte((word >> 24) & 0xFF, consts) << 24)
| (sbox_custom_byte((word >> 16) & 0xFF, consts) << 16)
| (sbox_custom_byte((word >> 8) & 0xFF, consts) << 8)
| sbox_custom_byte(word & 0xFF, consts)
)

def t_prime(word: int, consts: Constants) -> int:
x = tau(word, consts)
return (x ^ rotl32(x, 15) ^ rotl32(x, 23) ^ 0xCAFEBABE) & MASK32

def t_func(word: int, consts: Constants) -> int:
x = tau(word, consts)
return (x ^ rotl32(x, 3) ^ rotl32(x, 11) ^ rotl32(x, 19) ^ rotl32(x, 27) ^
0x12345678) & MASK32

def key_schedule(consts: Constants) -> List[int]:
mk = [int.from_bytes(consts.key_bytes[i:i + 4], "big") for i in range(0,
16, 4)]
rk = [0] * 32
b = [((mk[i] ^ consts.fk[i]) + i) & MASK32 for i in range(4)]

rk[0] = (b[0] ^ t_prime(b[1] ^ b[2] ^ b[3] ^ consts.ck[0], consts)) &
MASK32
rk[1] = (b[1] ^ t_prime(b[2] ^ b[3] ^ rk[0] ^ consts.ck[1], consts)) &
MASK32
rk[2] = (b[2] ^ t_prime(b[3] ^ rk[0] ^ rk[1] ^ consts.ck[2], consts)) &
MASK32
rk[3] = (b[3] ^ t_prime(rk[0] ^ rk[1] ^ rk[2] ^ consts.ck[3], consts)) &
MASK32

for i in range(4, 32):
rk[i] = ((rk[i - 4] ^ t_prime(rk[i - 3] ^ rk[i - 2] ^ rk[i - 1] ^
consts.ck[i], consts)) + i) & MASK32

return rk

def round_f(a: int, b: int, c: int, d: int, rk: int, consts: Constants) -> int:
return ((a ^ t_func(b ^ c ^ d ^ rk, consts)) + 0x1337) & MASK32

def words_from_bytes_be(block16: bytes) -> List[int]:
return [int.from_bytes(block16[i:i + 4], "big") for i in range(0, 16, 4)]

def words_to_bytes_be(words: Sequence[int]) -> bytes:
return b"".join((w & MASK32).to_bytes(4, "big") for w in words)

def block_decrypt(block16: bytes, consts: Constants, rk: Sequence[int]) ->
bytes:
y0, y1, y2, y3 = words_from_bytes_be(block16)

# Undo final affine swap/xor.
x = [
(y3 ^ 0x87654321) & MASK32,

(y2 ^ 0x10FEDCBA) & MASK32,
(y1 ^ 0xABCDEF01) & MASK32,
(y0 ^ 0x12345678) & MASK32,
]

for rnd in range(33, -1, -1):
if rnd in (8, 16, 24):
x[0] ^= 0x55555555
x[1] ^= 0xAAAAAAAA
x[0] &= MASK32
x[1] &= MASK32

b, c, d, e = x
a = (((e - 0x1337) & MASK32) ^ t_func(b ^ c ^ d ^ rk[rnd & 31],
consts)) & MASK32
x = [a, b, c, d]

x = [((w ^ 0xAAAAAAAA) & MASK32) for w in x]
return words_to_bytes_be(x)

def decrypt_final_target(consts: Constants) -> bytes:
rk = key_schedule(consts)
out = bytearray()
for i in range(0, 64, 16):
out.extend(block_decrypt(consts.target[i:i + 16], consts, rk))
return bytes(out)

def inverse_second_layer(buf90: bytes, ctx: Context, consts: Constants) ->
List[int]:
buf30 = [0] * 64
for i in range(64):
idx = ctx.tbl28[i] & 0x3F
t = buf90[i] ^ ctx.tbl20[i]
t = consts.aes_inv[t]
t ^= ctx.tbl30[i % 48]
buf30[idx] = t & 0xFF
return buf30

def round_r_values(ctx: Context) -> List[int]:
vals: List[int] = []
st = ctx.state[:]
for _rnd in range(6):
r, st = prng_next(st)
vals.append(r & 0x3F)

return vals

def full_round_transform_byte(x: int, pos: int, round_vals: Sequence[int],
aes_sbox: Sequence[int]) -> int:
for rnd, r in enumerate(round_vals):
x ^= (r + pos + rnd) & 0xFF
x = rol8(x, 1)
x ^= aes_sbox[(x + 13 * rnd) & 0xFF]
return x & 0xFF

def invert_first_transform(buf30: Sequence[int], ctx: Context, consts:
Constants) -> List[List[int]]:
round_vals = round_r_values(ctx)
mask = [((ctx.tbl20[(7 * i) & 0x3F] + i) & 0xFF) for i in range(64)]

candidates: List[List[int]] = []
for pos in range(64):
inv_map: Dict[int, List[int]] = {}
for x in range(256):
y = full_round_transform_byte(x, pos, round_vals, consts.aes_sbox)
inv_map.setdefault(y, []).append(x)

pre_round = inv_map.get(buf30[pos], [])
plaintext = sorted({b ^ mask[pos] for b in pre_round})
candidates.append(plaintext)

return candidates

def filter_candidates(
candidates: Sequence[Sequence[int]],
prefix: str,
suffix: str,
allowed: str,
) -> List[List[int]]:
allowed_set = {ord(c) for c in allowed}
filtered: List[List[int]] = []

for i, cands in enumerate(candidates):
cs = set(cands)

if i < len(prefix):
cs &= {ord(prefix[i])}

if suffix and i >= len(candidates) - len(suffix):

cs &= {ord(suffix[i - (len(candidates) - len(suffix))])}

cs &= allowed_set
filtered.append(sorted(cs))

return filtered

def enumerate_strings(filtered: Sequence[Sequence[int]], max_bruteforce: int =
100000) -> List[str]:
ambiguous = [i for i, cands in enumerate(filtered) if len(cands) > 1]
fixed = [cands[0] if len(cands) == 1 else None for cands in filtered]

if any(len(cands) == 0 for cands in filtered):
return []

total = 1
for i in ambiguous:
total *= len(filtered[i])
if total > max_bruteforce:
raise RuntimeError(
f"too many candidate combinations ({total}); tighten constraints
```

or inspect candidate sets manually"

```python
)

results: List[str] = []
for picks in product(*[filtered[i] for i in ambiguous]):
arr = fixed[:]
for pos, val in zip(ambiguous, picks):
arr[pos] = val
results.append(bytes(arr).decode("ascii", errors="replace"))
return results

def forward_validate_candidate(candidate: str, consts: Constants, ctx:
Context) -> bool:
"""Optional sanity-check: all survivors should reproduce the same
```

target."""

```python
if len(candidate) != 64:
return False

data = candidate.encode("ascii")
buf30 = []
for i in range(64):
x = data[i] ^ ((ctx.tbl20[(7 * i) & 0x3F] + i) & 0xFF)
buf30.append(x & 0xFF)

round_vals = round_r_values(ctx)
for pos in range(64):
buf30[pos] = full_round_transform_byte(buf30[pos], pos, round_vals,
consts.aes_sbox)

buf90 = [0] * 64
for i in range(64):
idx = ctx.tbl28[i] & 0x3F
t = buf30[idx] ^ ctx.tbl30[i % 48]
t = consts.aes_sbox[t]
t ^= ctx.tbl20[i]
buf90[i] = t & 0xFF

rk = key_schedule(consts)
out = bytearray()
for i in range(0, 64, 16):
block = bytes(buf90[i:i + 16])
# Reuse decrypt helper logic by re-implementing encrypt locally.
words = words_from_bytes_be(block)
x = [((w ^ 0xAAAAAAAA) & MASK32) for w in words]
for rnd in range(34):
new = round_f(x[0], x[1], x[2], x[3], rk[rnd & 31], consts)
x = [x[1], x[2], x[3], new]
if rnd in (8, 16, 24):
x[0] ^= 0x55555555
x[1] ^= 0xAAAAAAAA
x[0] &= MASK32
x[1] &= MASK32
final_words = [
(x[3] ^ 0x12345678) & MASK32,
(x[2] ^ 0xABCDEF01) & MASK32,
(x[1] ^ 0x10FEDCBA) & MASK32,
(x[0] ^ 0x87654321) & MASK32,
]
out.extend(words_to_bytes_be(final_words))

return bytes(out) == consts.target

def main() -> None:
ap = argparse.ArgumentParser(description="Recover the intended flag
directly from seg1_fixed_all.elf")
ap.add_argument("elf", type=Path, help="path to seg1_fixed_all.elf")
ap.add_argument("--prefix", default="flag{", help="expected flag prefix
(default: flag{)")
ap.add_argument("--suffix", default="}", help="expected flag suffix
(default: })")

ap.add_argument("--allowed", default=DEFAULT_ALLOWED, help="allowed
character set used to resolve ambiguities")
ap.add_argument("--show-candidates", action="store_true", help="print per-
position candidate characters before filtering")
args = ap.parse_args()

consts = load_constants(args.elf)
ctx = init_ctx(consts)
buf90 = decrypt_final_target(consts)
buf30 = inverse_second_layer(buf90, ctx, consts)
candidates = invert_first_transform(buf30, ctx, consts)

if args.show_candidates:
print("[*] raw candidate bytes per position:")
for i, cands in enumerate(candidates):
pretty = "".join(chr(c) if 32 <= c < 127 else "." for c in cands)
print(f" {i:02d}: {cands} {pretty}")
print()

filtered = filter_candidates(candidates, args.prefix, args.suffix,
args.allowed)
if any(len(c) == 0 for c in filtered):
raise SystemExit("[!] no candidates remain after applying
prefix/suffix/charset constraints")

results = enumerate_strings(filtered)
results = [r for r in results if forward_validate_candidate(r, consts,
ctx)]

if not results:
raise SystemExit("[!] no candidate survived forward validation")

if len(results) == 1:
print(results[0])
return

print("[!] multiple valid candidates remain:")
for r in results:
print(r)

if __name__ == "__main__":
main()
```

## 方法总结
- 核心技巧：固件容器解包 + ELF 修复 + 协议逆向
- 识别信号：文件有异常高频字节、魔数被扰动、容器内嵌压缩段和破坏 ELF。
- 复用要点：先还原外层编码和容器，再修 ELF header/segment/TLS，最后逆网络校验算法写客户端求解。
