# SU_Revird

## 题目简述
题目要求 `Complete the challenge.`，表面 `Check` 是假逻辑，真正链路是多层 Windows 逆向。外层程序先用魔改 AES 解出内嵌 EXE；该 EXE 读取输入后打开 `\\.\Revird` 设备，与驱动交互完成真正的加密/校验流程。

关键机制是用户态和驱动态共同实现一个 AES-like 变换：用户态负责 key expansion、LCG 随机表、IV/明文预处理和发包，驱动的多个 `op` case 分别完成轮密钥异或、ShiftRows、MixColumns 和字节级 S-box 化简。解题需要把两侧变换合并后反推最终输入，而不是只分析外层 `Check`。

## 解题过程
Main 函数里的 `Check` 是假逻辑，真正关键点在另一个调用链：程序用魔改 AES 解密出一段内嵌数据，动态调试后可确认解密结果是一个 EXE。提取脚本如下：

```python
from idaapi import get_byte

addr = 0x153C7717880
data = bytes(get_byte(addr + i) for i in range(0x4410))
open("2.exe", "wb").write(data)
```

提取出的用户态程序会读取输入，打开 `\\.\Revird` 设备，再把本地计算和驱动返回结果组合成最终校验。用户态部分包含 AES key expansion、LCG 生成的 256 字节随机表，以及与驱动通信前对 IV/明文的预处理。

驱动 `.sys` 中最关键的是几个 `op` 分支：

```text
op = 5: state ^= driver_roundkey[0]
op = 3: ShiftRows
op = 4: MixColumns 后再异或 driver_roundkey[round]
op = 6: state ^= driver_roundkey[10]
op = 2: 字节级 S-box 化简分支
```

`op = 2` 是整题的关键。驱动逻辑看似复杂，实际发包前用户态已经构造：

```text
data0 = state
data1 = state ^ G
```

驱动内部再计算：

```text
data1[i] ^ G[i] = (state[i] ^ G[i]) ^ G[i] = state[i]
driver_out2[i] = table_t[state[i]]
```

用户态收包后还会做：

```text
new_state[i] = driver_out2[i] ^ rand_table[state[i]]
```

所以整个 `op = 2` 的有效效果就是一个纯字节级 S-box：

```text
S[x] = table_t[x] ^ rand_table[x]
```

把用户态和驱动态的变换合并后，加密流程可以整理为：

```text
state ^= worker_roundkey[0]
state ^= driver_roundkey[0]

for round = 1..9:
    state = SBox(state)
    state = ShiftRows(state)   # 驱动 op=3
    state = ShiftRows(state)   # worker 本地再做一次
    state = MixColumns(state)  # 驱动 op=4 的前半段
    state ^= driver_roundkey[round]
    state ^= worker_roundkey[round]

final round:
    state = SBox(state)
    state = ShiftRows(state)
    state = ShiftRows(state)
    state ^= driver_roundkey[10]
    state ^= worker_roundkey[10]
```

### Exp

```python
from __future__ import annotations
import sys
from pathlib import Path

import pefile

def xtime(x: int) -> int:
x &= 0xFF
return (((x << 1) & 0xFF) ^ (0x1B if x & 0x80 else 0))

def mul(x: int, n: int) -> int:
r = 0
while n:
if n & 1:
r ^= x
x = xtime(x)
n >>= 1
return r & 0xFF

def mix_col(col: list[int]) -> list[int]:
a0, a1, a2, a3 = col
return [
mul(a0, 2) ^ mul(a1, 3) ^ a2 ^ a3,
a0 ^ mul(a1, 2) ^ mul(a2, 3) ^ a3,
a0 ^ a1 ^ mul(a2, 2) ^ mul(a3, 3),
mul(a0, 3) ^ a1 ^ a2 ^ mul(a3, 2),
]

def inv_mix_col(col: list[int]) -> list[int]:
a0, a1, a2, a3 = col
return [
mul(a0, 14) ^ mul(a1, 11) ^ mul(a2, 13) ^ mul(a3, 9),
mul(a0, 9) ^ mul(a1, 14) ^ mul(a2, 11) ^ mul(a3, 13),
mul(a0, 13) ^ mul(a1, 9) ^ mul(a2, 14) ^ mul(a3, 11),
mul(a0, 11) ^ mul(a1, 13) ^ mul(a2, 9) ^ mul(a3, 14),
]

def key_schedule(key: bytes, sbox: bytes, rcon: bytes) -> list[list[int]]:
if len(key) != 16:
raise ValueError("key must be 16 bytes")

sbox_list = list(sbox)

rcon_list = list(rcon)

def sub_word(word: list[int]) -> list[int]:
return [sbox_list[b] for b in word]

def rot_word(word: list[int]) -> list[int]:
return word[1:] + word[:1]

words: list[list[int]] = [list(key[i : i + 4]) for i in range(0, 16, 4)]
for i in range(4, 44):
temp = words[i - 1][:]
if i % 4 == 0:
temp = sub_word(rot_word(temp))
temp[0] ^= rcon_list[i // 4]
words.append([(words[i - 4][j] ^ temp[j]) & 0xFF for j in range(4)])
return [sum(words[r * 4 : (r + 1) * 4], []) for r in range(11)]

def shift_rows_once(state: list[int]) -> list[int]:
out = state[:]
out[1], out[5], out[9], out[13] = state[5], state[9], state[13], state[1]
out[2], out[6], out[10], out[14] = state[10], state[14], state[2], state[6]
out[3], out[7], out[11], out[15] = state[15], state[3], state[7], state[11]
return out

def shift_rows_twice(state: list[int]) -> list[int]:
# Exactly matches: driver opcode 3 + worker's local permutation.
return shift_rows_once(shift_rows_once(state))

def mix_columns(state: list[int]) -> list[int]:
out = [0] * 16
for c in range(4):
out[c * 4 : (c + 1) * 4] = mix_col(state[c * 4 : (c + 1) * 4])
return out

def inv_mix_columns(state: list[int]) -> list[int]:
out = [0] * 16
for c in range(4):
out[c * 4 : (c + 1) * 4] = inv_mix_col(state[c * 4 : (c + 1) * 4])
return out

def add_round_key(state: list[int], rk: list[int]) -> list[int]:
return [a ^ b for a, b in zip(state, rk)]

def build_effective_sbox(driver_img: bytes) -> tuple[list[int], list[int]]:
table_t = list(driver_img[0x3360 : 0x3360 + 256])

# Same 256-byte random table the worker generates with the fixed LCG seed.
seed = 0xC0FFEE13
rand_table: list[int] = []
for _ in range(256):
seed = (seed * 0x19660D + 0x3C6EF35F) & 0xFFFFFFFF
rand_table.append((seed >> 24) & 0xFF)

sbox_eff = [table_t[i] ^ rand_table[i] for i in range(256)]
inv_sbox_eff = [0] * 256
for i, b in enumerate(sbox_eff):
inv_sbox_eff[b] = i
return sbox_eff, inv_sbox_eff

class RevirdBlockCipher:
def __init__(self, worker_img: bytes, driver_img: bytes) -> None:
base_sbox = worker_img[0x42B0 : 0x42B0 + 256]
rcon = worker_img[0x43B0 : 0x43B0 + 16]
worker_key = worker_img[0x4400 : 0x4410]
driver_key = driver_img[0x3348 : 0x3358]

worker_rks = key_schedule(worker_key, base_sbox, rcon)
driver_rks = key_schedule(driver_key, base_sbox, rcon)

self.round_keys = [
[a ^ b for a, b in zip(worker_rks[r], driver_rks[r])]
for r in range(11)
]

self.sbox_eff, self.inv_sbox_eff = build_effective_sbox(driver_img)

def _sub_bytes(self, state: list[int]) -> list[int]:
return [self.sbox_eff[b] for b in state]

def _inv_sub_bytes(self, state: list[int]) -> list[int]:
return [self.inv_sbox_eff[b] for b in state]

def encrypt_block(self, block: bytes) -> bytes:
if len(block) != 16:
raise ValueError("block must be 16 bytes")

state = list(block)

state = add_round_key(state, self.round_keys[0])
for r in range(1, 10):
state = self._sub_bytes(state)
state = shift_rows_twice(state)
state = mix_columns(state)
state = add_round_key(state, self.round_keys[r])
state = self._sub_bytes(state)
state = shift_rows_twice(state)
state = add_round_key(state, self.round_keys[10])
return bytes(state)

def decrypt_block(self, block: bytes) -> bytes:
if len(block) != 16:
raise ValueError("block must be 16 bytes")

state = list(block)
state = add_round_key(state, self.round_keys[10])
state = shift_rows_twice(state) # self-inverse
state = self._inv_sub_bytes(state)
for r in range(9, 0, -1):
state = add_round_key(state, self.round_keys[r])
state = inv_mix_columns(state)
state = shift_rows_twice(state) # self-inverse
state = self._inv_sub_bytes(state)
state = add_round_key(state, self.round_keys[0])
return bytes(state)

def recover_flag(worker_path: Path, driver_path: Path) -> bytes:
wpe = pefile.PE(str(worker_path))
dpe = pefile.PE(str(driver_path))
worker_img = wpe.get_memory_mapped_image()
driver_img = dpe.get_memory_mapped_image()

cipher = RevirdBlockCipher(worker_img, driver_img)
iv = bytes(worker_img[0x4410 : 0x4420])
target = bytes(worker_img[0x43C0 : 0x4400])

# Decrypt CBC.
plaintext = bytearray()
prev = iv
for i in range(0, len(target), 16):
c = target[i : i + 16]
p = cipher.decrypt_block(c)
plaintext.extend(a ^ b for a, b in zip(p, prev))
prev = c

# Remove PKCS#7 padding.
pad = plaintext[-1]
if not 1 <= pad <= 16 or plaintext[-pad:] != bytes([pad]) * pad:
raise RuntimeError("invalid PKCS#7 padding after CBC decryption")
return bytes(plaintext[:-pad])

def main() -> None:
if len(sys.argv) != 3:
print("Usage: python decrypt_revird_flag.py <embedded_checker.exe>
<Revird.sys>")
raise SystemExit(1)

worker_path = Path(sys.argv[1])
driver_path = Path(sys.argv[2])
flag = recover_flag(worker_path, driver_path)
print(flag.decode("utf-8"))

if __name__ == "__main__":
main()
```

## 方法总结
- 核心技巧：用户态壳 + 驱动协同校验逆向
- 识别信号：主程序解出第二阶段并通过设备对象/IOCTL 与驱动交互。
- 复用要点：先动态/静态解出嵌入 EXE，再分析驱动 case 和通信参数，最后反向实现加密流程。
