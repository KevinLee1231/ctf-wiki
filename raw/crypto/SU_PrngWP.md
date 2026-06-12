# SU_Prng

## 题目简述
附件是一个 LCG 派生输出题。`chal.py` 中内部状态按 `seed = seed*a+b mod m` 更新，输出不是完整状态，而是高低半部分异或后再按状态高位做 `ror(...)` 旋转；服务最终要求提交原始 seed，并用 `md5(str(seed))` 做验证。题目价值在于从非线性包装后的 PRNG 输出恢复 LCG 状态，WP 不需要写入全部输出数据，只需要说明状态恢复约束。

## 解题过程
参考https://tosc.iacr.org/index.php/ToSC/article/view/8700/8292

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import ast
import hashlib
import re
import socket
import subprocess
import sys
from dataclasses import dataclass
from pathlib import Path
from typing import Iterable

from sympy import Matrix

BITS = 256
OUTS = 56
MASK128 = (1 << 128) - 1
MASK256 = (1 << 256) - 1
MASK14 = (1 << 14) - 1
ROT_WINDOWS = (0, 12, 24, 36)
WEIGHT_M = 16
WEIGHT_COUNT = 3

WEIGHT_KBITS = 32

def rol(x: int, k: int, n: int = 256) -> int:
k %= n
return ((x << k) | (x >> (n - k))) & ((1 << n) - 1)

def ror(x: int, k: int, n: int = 256) -> int:
k %= n
return ((x >> k) | (x << (n - k))) & ((1 << n) - 1)

class Tube:
def __init__(self) -> None:
self._buf = bytearray()

def _recv_chunk(self) -> bytes:
raise NotImplementedError

def sendline(self, data: bytes) -> None:
raise NotImplementedError

def close(self) -> None:
raise NotImplementedError

def recv_until(self, token: bytes) -> bytes:
while token not in self._buf:
chunk = self._recv_chunk()
if not chunk:
raise EOFError(f"connection closed while waiting for
{token!r}")
self._buf.extend(chunk)
idx = self._buf.index(token) + len(token)
out = bytes(self._buf[:idx])
del self._buf[:idx]
return out

def recv_all(self) -> bytes:
while True:
chunk = self._recv_chunk()
if not chunk:
out = bytes(self._buf)
self._buf.clear()
return out
self._buf.extend(chunk)

class ProcessTube(Tube):
def __init__(self, argv: list[str], cwd: Path) -> None:
super().__init__()
self.proc = subprocess.Popen(
argv,
cwd=cwd,
stdin=subprocess.PIPE,
stdout=subprocess.PIPE,
stderr=subprocess.STDOUT,
)
assert self.proc.stdin is not None
assert self.proc.stdout is not None
self.stdin = self.proc.stdin
self.stdout = self.proc.stdout

def _recv_chunk(self) -> bytes:
return self.stdout.read1(4096)

def sendline(self, data: bytes) -> None:
self.stdin.write(data + b"\n")
self.stdin.flush()

def close(self) -> None:
if self.proc.poll() is None:
self.proc.kill()
self.proc.wait()

class SocketTube(Tube):
def __init__(self, host: str, port: int) -> None:
super().__init__()
self.sock = socket.create_connection((host, port))

def _recv_chunk(self) -> bytes:
return self.sock.recv(4096)

def sendline(self, data: bytes) -> None:
self.sock.sendall(data + b"\n")

def close(self) -> None:
self.sock.close()

@dataclass(frozen=True)
class RotationCandidate:
x1_low14: int

b_low14: int
rseq: tuple[int, ...]
zseq: tuple[int, ...]

@dataclass(frozen=True)
class LowHalfCandidate:
x1: int
b_low: int
rotation: RotationCandidate

WEIGHT_CACHE: dict[tuple[int, int, int, int, int], list[tuple[int, tuple[int,
...]]]] = {}

def candidate_rotations(y: int) -> list[int]:
return [r for r in range(256) if (rol(y, r) >> 128) == 0]

def recover_rotation_sequences(a: int, outputs: list[int]) ->
list[RotationCandidate]:
rot_cands = [candidate_rotations(y) for y in outputs]
a14 = a & MASK14
seen: set[tuple[int, int, tuple[int, ...]]] = set()
results: list[RotationCandidate] = []

for r0 in rot_cands[0]:
for low0 in range(64):
x0 = (r0 << 6) | low0
for r1 in rot_cands[1]:
for low1 in range(64):
x1 = (r1 << 6) | low1
b14 = (x1 - a14 * x0) & MASK14
xs = [x0, x1]
ok = True
for idx in range(2, len(outputs)):
xs.append((a14 * xs[-1] + b14) & MASK14)
if ((xs[-1] >> 6) & 0xFF) not in rot_cands[idx]:
ok = False
break
if not ok:
continue

rseq = tuple((x >> 6) & 0xFF for x in xs)
key = (x0, b14, rseq)
if key in seen:

continue
seen.add(key)
zseq = tuple(rol(outputs[i], rseq[i]) & MASK128 for i in
range(len(outputs)))
results.append(RotationCandidate(x0, b14, rseq, zseq))
return results

def weight_vectors(a_mod: int, mod_bits: int, m: int = WEIGHT_M, count: int =
WEIGHT_COUNT) -> list[tuple[int, tuple[int, ...]]]:
key = (a_mod, mod_bits, m, count, WEIGHT_KBITS)
if key in WEIGHT_CACHE:
return WEIGHT_CACHE[key]

modulus = 1 << mod_bits
scale = 1 << WEIGHT_KBITS
rows = [
[scale * pow(a_mod, power, modulus)] + [1 if col == power else 0 for
col in range(m)]
for power in range(m)
]
rows.append([scale * modulus] + [0] * m)

reduced = Matrix(rows).lll()
vectors: list[tuple[int, tuple[int, ...]]] = []
seen: set[tuple[int, ...]] = set()
for row in reduced.tolist():
if row[0] != 0 or not any(row[1:]):
continue
weights = tuple(int(v) for v in row[1:])
if weights in seen:
continue
seen.add(weights)
vectors.append((max(abs(v) for v in weights), weights))

vectors.sort(key=lambda item: item[0])
WEIGHT_CACHE[key] = vectors[:count]
return WEIGHT_CACHE[key]

def survives_filter(a: int, candidate: LowHalfCandidate, t: int) -> bool:
mask = (1 << t) - 1
needed = max(ROT_WINDOWS) + WEIGHT_M + 1
a_low = a & MASK128

xseq = [candidate.x1]
for _ in range(1, needed):

xseq.append((a_low * xseq[-1] + candidate.b_low) & mask)

modulus = 1 << (128 + t)
vectors = weight_vectors(a % modulus, 128 + t)

for start in ROT_WINDOWS:
approx_states = [
((((candidate.rotation.zseq[i] & mask) ^ xseq[i]) << 128) %
modulus)
for i in range(start, start + WEIGHT_M + 1)
]
for width, weights in vectors:
accum = 0
for j, coeff in enumerate(weights):
diff = (approx_states[j + 1] - approx_states[j]) % modulus
accum = (accum + coeff * diff) % modulus
dist = min(accum, modulus - accum)
bound = 2 * len(weights) * width * (1 << 128)
if dist > bound:
return False
return True

def recover_low_half_candidates(a: int, rotations: list[RotationCandidate],
verbose: bool = False) -> list[LowHalfCandidate]:
candidates: list[LowHalfCandidate] = []
for rotation in rotations:
for extra_x in range(4):
for extra_b in range(4):
cand = LowHalfCandidate(
rotation.x1_low14 | (extra_x << 14),
rotation.b_low14 | (extra_b << 14),
rotation,
)
if survives_filter(a, cand, 16):
candidates.append(cand)

if verbose:
print(f"[+] t=16 candidates: {len(candidates)}", file=sys.stderr)

t = 16
while t < 128:
step = min(4, 128 - t)
next_candidates: list[LowHalfCandidate] = []
seen: set[tuple[int, int, tuple[int, ...]]] = set()
for cand in candidates:
for extra_x in range(1 << step):

for extra_b in range(1 << step):
nxt = LowHalfCandidate(
cand.x1 | (extra_x << t),
cand.b_low | (extra_b << t),
cand.rotation,
)
key = (nxt.x1, nxt.b_low, nxt.rotation.rseq)
if key in seen:
continue
if survives_filter(a, nxt, t + step):
seen.add(key)
next_candidates.append(nxt)
candidates = next_candidates
t += step
if verbose:
print(f"[+] t={t} candidates: {len(candidates)}", file=sys.stderr)
if not candidates:
break
return candidates

def recover_seed_from_state(a: int, b: int, first_state: int, digest: str) ->
int | None:
rhs = (first_state - b) & MASK256
v2 = 0
aa = a
while v2 < 256 and (aa & 1) == 0:
aa >>= 1
v2 += 1

if rhs & ((1 << v2) - 1):
return None

modulus = 1 << (256 - v2)
inv = pow(aa, -1, modulus)
base = ((rhs >> v2) * inv) % modulus

if v2 > 20:
raise RuntimeError(f"too many seed lifts to enumerate: 2^{v2}")

for k in range(1 << v2):
candidate = base + (k << (256 - v2))
if 0 < candidate <= (1 << 256) and
hashlib.md5(str(candidate).encode()).hexdigest() == digest:
return candidate
return None

def verify_candidate(a: int, outputs: list[int], digest: str, candidate:
LowHalfCandidate) -> int | None:
z1 = candidate.rotation.zseq[0]
z2 = candidate.rotation.zseq[1]
first_state = (((z1 ^ candidate.x1) << 128) | candidate.x1) & MASK256
x2 = (((a & MASK128) * candidate.x1) + candidate.b_low) & MASK128
second_state = (((z2 ^ x2) << 128) | x2) & MASK256
b = (second_state - a * first_state) & MASK256

state = first_state
for y in outputs:
x = state & MASK128
z = (state >> 128) ^ x
if ror(z, (state >> 6) & 0xFF) != y:
return None
state = (a * state + b) & MASK256

return recover_seed_from_state(a, b, first_state, digest)

def solve_instance(a: int, outputs: list[int], digest: str, verbose: bool =
False) -> int:
rotations = recover_rotation_sequences(a, outputs)
if verbose:
print(f"[+] rotation sequences: {len(rotations)}", file=sys.stderr)
if not rotations:
raise RuntimeError("failed to recover a valid rotation sequence")

low_half_candidates = recover_low_half_candidates(a, rotations,
verbose=verbose)
if not low_half_candidates:
raise RuntimeError("failed to recover low-half candidates")

if verbose:
print(f"[+] final low-half candidates: {len(low_half_candidates)}",
file=sys.stderr)

for idx, candidate in enumerate(low_half_candidates, 1):
seed = verify_candidate(a, outputs, digest, candidate)
if seed is not None:
if verbose:
print(f"[+] candidate #{idx} verified", file=sys.stderr)
return seed
raise RuntimeError("no candidate matched the full output stream")

def parse_banner(text: str) -> tuple[int, list[int], str]:
match_a = re.search(r"a = (\d+)", text)
match_out = re.search(r"out = (\[[^\n]+\])", text)
match_h = re.search(r"h = ([0-9a-f]{32})", text)
if not (match_a and match_out and match_h):
raise RuntimeError(f"failed to parse challenge banner:\n{text}")
return int(match_a.group(1)), list(ast.literal_eval(match_out.group(1))),
match_h.group(1)

def make_tube(args: argparse.Namespace, base_dir: Path) -> Tube:
if args.remote:
host, port = args.remote
return SocketTube(host, port)
return ProcessTube([sys.executable, "-u", "chal.py"], base_dir)

def main() -> int:
parser = argparse.ArgumentParser(description="Exploit for SU_Prng")
parser.add_argument("--remote", nargs=2, metavar=("HOST", "PORT"),
help="connect to remote service")
parser.add_argument("--verbose", action="store_true")
args = parser.parse_args()

if args.remote:
args.remote = (args.remote[0], int(args.remote[1]))

base_dir = Path(__file__).resolve().parent
tube = make_tube(args, base_dir)
try:
banner = tube.recv_until(b"> ").decode(errors="replace")
a, outputs, digest = parse_banner(banner)
if len(outputs) != OUTS:
raise RuntimeError(f"unexpected output count: {len(outputs)}")

if args.verbose:
print(f"[+] parsed a and {len(outputs)} outputs", file=sys.stderr)

seed = solve_instance(a, outputs, digest, verbose=args.verbose)
print(f"[+] recovered seed: {seed}")

tube.sendline(str(seed).encode())
tail = (tube.recv_all() if not args.remote else
tube.recv_until(b"}")).decode(errors="replace")
sys.stdout.write(tail)
if not tail.endswith("\n"):
print()

return 0
finally:
tube.close()

if __name__ == "__main__":
raise SystemExit(main())
```

## 方法总结
- 核心技巧：LCG 状态恢复/隐藏数约束
- 识别信号：输出来自 LCG 状态的截断、异或和旋转变换，且有多轮连续输出。
- 复用要点：把输出关系转成对状态的约束，结合 LCG 递推恢复 seed；最后用服务给出的 hash 或回放输出验证。
