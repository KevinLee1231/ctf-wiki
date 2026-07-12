# SUCTF2026-West

## 题目简述
题目以西游记诗句作提示，要求输入 81 轮 16 位十进制数，每轮通过 dispatch table 中不同函数更新状态。关键结构包括 permutation、函数指针表和每个函数内部不同的常量表；解法是恢复状态结构和每轮变换，再用约束求解。

## 解题过程
一共输入 81 轮数据，每轮要求输入 16 位十进制数。主函数中会按 `permutation` 表选择当前轮的 dispatch 函数，输入不通过统一函数处理，而是每一轮走不同的函数和不同的 `0xc0` 字节常量块。

主循环可以整理成如下形式：

```c
for (round = 0; round < 81; round++) {
    layer = permutation[round];
    printf("%s | round %u %u: ", prompt[layer], round + 1, layer + 1);
    fgets(input, 128, stdin);
    nums[round] = parse_16_digit_decimal(input);
    if (!dispatch[layer](&state, nums[round])) {
        return 0;
    }
}
```

恢复出的状态结构大致为：

```c
struct State {
    uint64_t s0;       // +0x00
    uint64_t idx;      // +0x08
    uint64_t s2;       // +0x10
    uint32_t counter;  // +0x18
    uint8_t flag[40];  // +0x1c
};
```

整体调度结构如下：

```python
for round_idx in range(81):
    layer = perm[round_idx]
    blob = blob_table[layer]
    fn = dispatch[layer]
    fn(state, input[round_idx])
```

`sub_140001100` 不直接加密输入，主要用于更新 `State.s2` 并有条件地修改 `counter`。真正比较链在 dispatch 函数中，重点是三个可逆 helper；它们看起来很长，本质都是“固定旋转 + 加法 + xor”。

`sub_140012480` 的核心逻辑可以整理为：

```python
v = input_value ^ blob[5]
hi = v >> 32
lo = v & 0xffffffff

for each round i:
    k = ...  # 由 s0 / idx / layer / blob / 常量算出
    tmp = ((k_hi ^ lo) + rol32(lo ^ k_lo, rot_l)) ^ hi
    new_lo = (ror32(lo, rot_r) + k_lo) ^ tmp
    new_hi = lo
    hi, lo = new_hi, new_lo

out = (hi << 32) | lo
```

sub_140012630

```
state = ((idx * PHI + PHI) ^ input ^ (layer * DELTA) ^ blob[5] ^ s0)
seed_base = 0x94d049bb133111eb - idx * 0x6b2fb644ecceee15
acc = 0xbf58476d1ce4e5b9

for i in range(blob[a0] + 2):
rot = ((idx + blob[a3] + layer + i) % 63) + 1
state = rol64(state ^ seed_i ^ blob_q15_i, rot) + add_i
```

sub_140012940

```
out = x
for i in range(((blob[a3] + idx) & 1) + 3):
t = ...
v = rol64(blob[9], rot2)
u = rol64(blob[5], rot1) ^ (blob[0] + t)
out = rol64(t ^ out ^ v, rot3) + u

ret = ((idx * 0x94d049bb133111eb + const) ^ blob[10]) ^ out
```

当然只逆出加密逻辑是不够的因为State并不是一成不变的，所以需要配合unicorn模拟执行从而去推
进状态

### Exp:

```python
from __future__ import annotations

import argparse
import struct
from dataclasses import dataclass

import pefile

from unicorn import Uc, UC_ARCH_X86, UC_MODE_64
from unicorn.x86_const import (
UC_X86_REG_R8,
UC_X86_REG_R9,
UC_X86_REG_RAX,
UC_X86_REG_RCX,
UC_X86_REG_RDX,
UC_X86_REG_RIP,
UC_X86_REG_RSP,
)

def u32(x: int) -> int:
return x & 0xFFFFFFFF

def u64(x: int) -> int:
return x & 0xFFFFFFFFFFFFFFFF

def rol32(x: int, n: int) -> int:
x &= 0xFFFFFFFF
n &= 31
return u32(((x << n) | (x >> (32 - n))) if n else x)

def ror32(x: int, n: int) -> int:
x &= 0xFFFFFFFF
n &= 31
return u32(((x >> n) | (x << (32 - n))) if n else x)

def rol64(x: int, n: int) -> int:
x &= 0xFFFFFFFFFFFFFFFF
n &= 63
return u64(((x << n) | (x >> (64 - n))) if n else x)

def ror64(x: int, n: int) -> int:
x &= 0xFFFFFFFFFFFFFFFF
n &= 63
return u64(((x >> n) | (x << (64 - n))) if n else x)

DELTA = 0xA24BAED4963EE407
PHI = 0x9E3779B97F4A7C15
MIX2 = 0xD6E8FEB86659FD93

CONST126 = 0x6B2FB644ECCEEE15
C_BF = 0xBF58476D1CE4E5B9
C_129F = 0x94D049BB133111EB

@dataclass
class ImageCtx:
pe: pefile.PE
base: int

def q(self, blob: bytes, i: int) -> int:
return struct.unpack_from("<Q", blob, i * 8)[0]

def inv12480(ctx: ImageCtx, out: int, s0: int, idx: int, layer: int, blob:
bytes) -> int:
"""Inverse of helper 0x12480 for this blob/round/layer."""

k0 = u64((idx * PHI + PHI) ^ s0 ^ (layer * MIX2) ^ ctx.q(blob, 0))
const_a = u32(blob[0xA2] + idx + 7)
cur_b = u32(blob[0xA2] + idx + 6)
cur_c = u32(blob[0xA2] + idx)
const_d = u32(blob[0xA2] + idx + 1)
count = blob[0xA1] + 6

rounds: list[tuple[int, int, int]] = []
delta = DELTA
cb = cur_b
cc = cur_c
for i in range(count):
q1 = cc // 31
ecx = u32(31 * q1)
q2 = u32(cb - ecx) // 31
rot_r = u32(const_a - ecx - 31 * q2 + i)
rot_l = u32(const_d - ecx + i)
k = u64(k0 ^ delta ^ ctx.q(blob, 1 + (i & 3)))
rounds.append((k, rot_l, rot_r))
delta = u64(delta + DELTA)
cb = u32(cb + 1)
cc = u32(cc + 1)

high = (out >> 32) & 0xFFFFFFFF
low = out & 0xFFFFFFFF

for k, rot_l, rot_r in reversed(rounds):
new_high, new_low = high, low
old_low = new_high

tmp = u32(ror32(old_low, rot_r) + (k & 0xFFFFFFFF))
tmp ^= new_low

old_high = u32(((k >> 32) & 0xFFFFFFFF) ^ old_low)
old_high = u32(old_high + rol32(u32(old_low ^ (k & 0xFFFFFFFF)),
rot_l))
old_high ^= tmp

high, low = old_high, old_low

v = u64((high << 32) | low)
return u64(v ^ ctx.q(blob, 5))

def inv12630(ctx: ImageCtx, out: int, s0: int, idx: int, layer: int, blob:
bytes) -> int:
"""Inverse of helper 0x12630 for this blob/round/layer."""

b3 = blob[0xA3]
cur0 = idx + b3 + layer
seed_base = u64(0x94D049BB133111EB - u64(idx * CONST126))

rounds: list[tuple[int, int, int]] = []
seed = seed_base
acc = C_BF
count = blob[0xA0] + 2
for i in range(count):
rot = ((cur0 + i) % 63) + 1
xor_const = u64(seed ^ ctx.q(blob, 15 + ((b3 + i) & 3)))
add_const = u64(ctx.q(blob, 11 + (i & 3)) ^ acc ^ s0)
rounds.append((rot, xor_const, add_const))
seed = u64(seed + seed_base)
acc = u64(acc + C_BF)

state = out
for rot, xor_const, add_const in reversed(rounds):
state = u64(state - add_const)
state = ror64(state, rot)
state ^= xor_const

x = state ^ u64((u64(idx * PHI + PHI)) ^ (layer * DELTA) ^ ctx.q(blob, 5)
^ s0)
return u64(x)

def inv12940(ctx: ImageCtx, out: int, idx: int, blob: bytes) -> int:

"""Inverse of helper 0x12940 for this blob/round."""

b3 = blob[0xA3]
b2 = blob[0xA2]
count = ((b3 + idx) & 1) + 3
delta_base = u64(idx * DELTA + DELTA)
delta = delta_base

rounds: list[tuple[int, int, int, int]] = []
for i in range(count):
t = u64(
ctx.q(blob, 8)
^ delta
^ ctx.q(blob, 7)
^ ctx.q(blob, 1 + i)
^ ctx.q(blob, 11 + ((b3 + i) & 3))
)
rot2 = ((b3 + i) % 63) + 1
rot1 = ((b2 + i) % 63) + 1
rot3 = ((idx + b2 + 3 * i) % 63) + 1
v = rol64(ctx.q(blob, 9), rot2)
u = u64(rol64(ctx.q(blob, 5), rot1) ^ u64(ctx.q(blob, 0) + t))
rounds.append((t, v, u, rot3))
delta = u64(delta + delta_base)

x = u64(out ^ (u64(idx * C_129F + C_129F) ^ ctx.q(blob, 10)))
for t, v, u, rot3 in reversed(rounds):
x = u64(x - u)
x = ror64(x, rot3)
x ^= t ^ v
return x

def solve_input_for_round(ctx: ImageCtx, s0: int, idx: int, layer: int, blob:
bytes) -> int:
target = ctx.q(blob, 19)
y = inv12940(ctx, target, idx, blob)
x1 = inv12630(ctx, y, s0, idx, layer, blob)
inp = inv12480(ctx, x1, s0, idx, layer, blob)
return inp

class Emulator:
def __init__(self, pe: pefile.PE) -> None:
self.pe = pe
self.base = pe.OPTIONAL_HEADER.ImageBase
self.size = pe.OPTIONAL_HEADER.SizeOfImage

self.img = pe.get_memory_mapped_image()[: self.size]

self.stack = 0x7000000000
self.stack_size = 0x200000
self.ret = 0x6000000000
self.scratch = 0x5000000000
self.state_ptr = self.scratch + 0x1000

self.uc = Uc(UC_ARCH_X86, UC_MODE_64)
map_base = self.base & ~0xFFF
map_size = (self.size + (self.base - map_base) + 0xFFF) & ~0xFFF
self.uc.mem_map(map_base, map_size)
self.uc.mem_write(self.base, self.img)
self.uc.mem_map(self.stack, self.stack_size)
self.uc.mem_map(self.ret, 0x1000)
self.uc.mem_map(self.scratch, 0x100000)

# Two environment-dependent helpers are only used in the wrapper path.
# Replacing them with tiny stubs keeps emulation deterministic and does
# not affect the core compare chain we reversed (12480/12630/12940).
self.uc.mem_write(self.base + 0x1780, b"\x31\xC0\xC3") # xor eax,eax
; ret
self.uc.mem_write(self.base + 0x16F94, b"\x31\xC0\xC3") # xor eax,eax
; ret

def call(self, addr: int, rcx: int = 0, rdx: int = 0, r8: int = 0, r9: int
= 0, stack_args: list[int] | None = None) -> int:
if stack_args is None:
stack_args = []
rsp = (self.stack + self.stack_size // 2) & ~0xF
layout = bytearray(0x1000)
struct.pack_into("<Q", layout, 0, self.ret)
off = 0x28 # Win64: return addr + 0x20 shadow + extra args
for arg in stack_args:
struct.pack_into("<Q", layout, off, arg & 0xFFFFFFFFFFFFFFFF)
off += 8
self.uc.mem_write(rsp, bytes(layout))
regs = [
(UC_X86_REG_RSP, rsp),
(UC_X86_REG_RCX, rcx),
(UC_X86_REG_RDX, rdx),
(UC_X86_REG_R8, r8),
(UC_X86_REG_R9, r9),
(UC_X86_REG_RIP, addr),
]
for reg, val in regs:
self.uc.reg_write(reg, val)

self.uc.emu_start(addr, self.ret)
return self.uc.reg_read(UC_X86_REG_RAX)

def write_initial_state(self) -> None:
flag0 = bytes.fromhex(
"8f129c59d5e29d23988f2bd108f36af0"
"634a3710306f3397c6d2c07b722582ff"
"cf5b9109c7141c78"
)
state = struct.pack(
"<QQQI",
0x669E1E61279D826E,
0,
0xA03AB9F27C4C6BFB,
0,
) + flag0
self.uc.mem_write(self.state_ptr, state)
self.call(self.base + 0x1D10)

def read_state(self) -> bytes:
return bytes(self.uc.mem_read(self.state_ptr, 8 * 3 + 4 + 40))

def write_round_index(self, round_idx: int) -> None:
st = bytearray(self.read_state())
struct.pack_into("<Q", st, 8, round_idx)
self.uc.mem_write(self.state_ptr, bytes(st))

def run_round(self, round_idx: int, layer: int, layer_fn: int, num: int) -
> None:
r = self.call(
self.base + 0x1100,
rcx=self.state_ptr,
rdx=round_idx,
r8=layer,
r9=num,
stack_args=[0xF00DFACECAFEBEEF, 0],
)
if r != 0:
raise RuntimeError(f"pre-wrapper failed at round {round_idx}: {r}")

r = self.call(layer_fn, rcx=self.state_ptr, rdx=num)
if r != 1:
raise RuntimeError(f"layer failed at round {round_idx}: {r}")

st_mid = self.read_state()
s0_after = struct.unpack_from("<Q", st_mid, 0)[0]
r = self.call(

self.base + 0x1100,
rcx=self.state_ptr,
rdx=round_idx,
r8=layer,
r9=u64(num ^ s0_after),
stack_args=[0xDEADC0DE12345678, 0],
)
if r != 0:
raise RuntimeError(f"post-wrapper failed at round {round_idx}:
{r}")

def main() -> None:
patch = "Journey_to_the_West.exe"

pe = pefile.PE(patch)
ctx = ImageCtx(pe=pe, base=pe.OPTIONAL_HEADER.ImageBase)
emu = Emulator(pe)
emu.write_initial_state()

perm = list(pe.get_data(0x3DEE0, 81))
dispatch = [struct.unpack_from("<Q", pe.get_data(0x2A480 + i * 8, 8))[0]
for i in range(81)]

nums: list[int] = []
for round_idx in range(81):
layer = perm[round_idx]
blob = pe.get_data(0x2A710 + 0xC0 * layer, 0xC0)
st = emu.read_state()
s0 = struct.unpack_from("<Q", st, 0)[0]
emu.write_round_index(round_idx)

num = solve_input_for_round(ctx, s0, round_idx, layer, blob)
if not (10**15 <= num <= 10**16 - 1):
raise RuntimeError(f"round {round_idx}: got non-16-digit value
{num}")

nums.append(num)
emu.run_round(round_idx, layer, dispatch[layer], num)
print(f"{round_idx:02d} layer={layer:02d} num={num}")

final_state = emu.read_state()
flag_bytes = final_state[28 : 28 + 40]
counter = struct.unpack_from("<I", final_state, 24)[0]

print("CSV=" + ",".join(str(x) for x in nums))
print("FLAG=" + flag_bytes.rstrip(b"\x00").decode("ascii", "replace"))

print(f"COUNTER={counter}")

if __name__ == "__main__":
main()
```

## 方法总结
- 核心技巧：大量分发函数的状态机反解
- 识别信号：输入轮数固定，函数指针表/置换表决定每轮变换。
- 复用要点：恢复结构体和调度表后，把每轮输入到状态更新写成约束，用脚本或 SMT 求解。
