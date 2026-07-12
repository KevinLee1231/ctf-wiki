# SUCTF2026-MirrorBus9

## 题目简述
题目是一个 half-duplex industrial bus 协议交互。核心命令 `RESET/ENQ MIX/ARM/COMMIT/POLL/PROVE` 可被建模为有限域上的仿射变换；题干提示 `PROVE` 校验的是 `CHAL` 帧而不是自己提交的 ARM 状态，前两个参数来自 `CHAL`，第三个参数是包含 nonce 的 16 位校验和。解法要先通过采样基点学习 ARM 变换，再转向 CHAL 帧校验，最终构造能通过 checksum 的参数。

## 解题过程
服务名是 MirrorBus-9 ，题面强调它是一个 half-duplex industrial bus。实际靶机交互里最关键
的命令有：

- RESET
- ENQ MIX a b c
- ARM
- COMMIT
- POLL
- PROVE p1 p2 p3

前期结论

前期分析可以确认这些事实：

- `RESET` 会把当前 TCP session 恢复到一个确定性的初始种子。
- `MIX a b c; ARM` 的第二帧在 `F_65521` 上是仿射的。

- 只要采 4 个基点：
  - `(0,0,0)`
  - `(1,0,0)`
  - `(0,1,0)`
  - `(0,0,1)`

就能恢复 reset 态下 ARM_FAIL 的仿射映射。

- 解

$$
B_arm + M_arm * x = 0
$$

可以得到一条 1 维直线，这条线上的点都能触发 CHAL 。

也就是说，ARM 部分本质上已经能解：

1. 先学 reset 态下的 ARM 仿射映射。

2. 再求出一组能过 ARM 的 MIX 参数。

真正卡住的点

最开始一直把 PROVE 当成“验证当前 ARM 状态”，所以做了很多围绕 active ARM line 的搜索，包
括：

- 直接把 ARM 线上的点拿去 PROVE

- 用 sig/aux/nonce 做各种线性组合

- 针对 active line、aligned line、若干 target family 做全空间扫描

这些都不对。

真正有用的 hint 是：

PROVE verifies the CHAL frame, not the ARM state you fed into it;

$$
the first two parameters are taken from CHAL , and the third is a 16-bit checksum that
$$
includes the nonce.

这句话一出来，题目就从“猜 3 维响应公式”直接降成了：

直接来自 CHAL • p1 、p2

- 只剩 p3 这个 16-bit 值未知

关键转折

CHAL 的内容形如：

```
F cid=1 tick=1 lane=0 sig=60056 aux=41938 tag=CHAL nonce=175a6f7bf012 ttl=192
```

根据 hint，可以确定：
- p1 = chal.sig
- p2 = chal.aux

是一个和 nonce 有关的 16-bit checksum• p3

虽然我补了很多常见 checksum/CRC16 候选去试：
- crc16_modbus
- crc16_x25

- crc16_ccitt
- crc16_xmodem
- fletcher16
- internet checksum

- 各种 sum16

- 文本帧 / 二进制帧 / 带 cid/tick/lane/ttl 的不同打包方式

但都没有直接命中。

这时候最稳的做法就不是继续猜公式，而是直接爆破 p3 。

为什么直接爆破可行

虽然 PROVE 每次 challenge 只能错 7 次，但这题有两个非常关键的性质：

1. RESET 会把同一个 TCP session 恢复到完全相同的初始 challenge。

2. 初始 CHAL 的 sig/aux/nonce 在同一个 session 里是固定的。

所以在同一个 TCP session 里，我们可以反复这样做：

```
RESET
ENQ MIX <valid_commit_point>
ARM
COMMIT
PROVE sig aux p3_0
PROVE sig aux p3_1
...
PROVE sig aux p3_6
```

一轮试 7 个 p3 ，错满了就再 RESET ，继续试下一批。

因为 reset 之后 challenge 还是同一个，所以这就等价于在同一个固定目标上做 16-bit 穷举。

利用脚本

```python
import argparse
import binascii
import re
import socket
from dataclasses import dataclass

HOST = "<target>"

MOD = 65521

@dataclass
class Frame:
cid: int
tick: int
lane: int
sig: int
aux: int
tag: str
rest: str
raw: str

FRAME_RE = re.compile(
r"^F cid=(\-?\d+) tick=(\-?\d+) lane=(\-?\d+) sig=(\-?\d+) aux=(\-?\d+)
tag=([^\s]+)(?:\s+(.*))?$"
)
NONCE_RE = re.compile(r"nonce=([0-9a-f]+)")

class MB9:
def __init__(self, port: int = 10011, timeout: float = 0.25):
self.port = port
self.timeout = timeout
self.s = socket.socket()
self.s.settimeout(3)
self.s.connect((HOST, port))
self.banner = self.recv_all(timeout=0.3)
m = re.search(r"sid=([0-9a-f]+)", self.banner)
self.sid = m.group(1) if m else None

def recv_all(self, timeout: float | None = None) -> str:
self.s.settimeout(self.timeout if timeout is None else timeout)
chunks = []
while True:
try:
data = self.s.recv(65535)
if not data:
break
chunks.append(data)
except socket.timeout:
break
return b"".join(chunks).decode("utf-8", "replace")

def batch(self, lines: list[str], timeout: float | None = None) -> str:
payload = "".join(line.rstrip("\n") + "\n" for line in lines)
self.s.sendall(payload.encode())
return self.recv_all(timeout)

def send(self, line: str, timeout: float | None = None) -> str:
self.s.sendall((line.rstrip("\n") + "\n").encode())
return self.recv_all(timeout)

def close(self) -> None:
try:
self.send("QUIT")
except OSError:
pass
self.s.close()

def parse_frames(text: str) -> list[Frame]:
out: list[Frame] = []
for line in text.splitlines():
m = FRAME_RE.match(line)
if not m:
continue
out.append(
Frame(
cid=int(m.group(1)),
tick=int(m.group(2)),
lane=int(m.group(3)),
sig=int(m.group(4)) % MOD,
aux=int(m.group(5)) % MOD,
tag=m.group(6),
rest=m.group(7) or "",
raw=line,
)
)
return out

def solve_affine_line(
base_sig: int,
base_aux: int,
row_sig: tuple[int, int, int],
row_aux: tuple[int, int, int],
c: int,
) -> tuple[int, int, int]:
det = (row_sig[0] * row_aux[1] - row_sig[1] * row_aux[0]) % MOD
rhs_sig = (-base_sig - row_sig[2] * c) % MOD
rhs_aux = (-base_aux - row_aux[2] * c) % MOD
inv_det = pow(det, -1, MOD)
a = (rhs_sig * row_aux[1] - row_sig[1] * rhs_aux) % MOD * inv_det % MOD
b = (row_sig[0] * rhs_aux - rhs_sig * row_aux[0]) % MOD * inv_det % MOD
return a, b, c % MOD

def learn_reset_maps(
mb: MB9,
) -> tuple[
tuple[int, int],
tuple[int, int, int],
tuple[int, int, int],
tuple[int, int],
tuple[int, int, int],
tuple[int, int, int],
]:
samples: dict[tuple[int, int, int], tuple[tuple[int, int], tuple[int,
int]]] = {}
for point in [(0, 0, 0), (1, 0, 0), (0, 1, 0), (0, 0, 1)]:
a, b, c = point
raw = mb.batch(["RESET", f"ENQ MIX {a} {b} {c}", "ARM", "COMMIT",
"POLL 8"], timeout=0.7)
frames = parse_frames(raw)
obs = (frames[0].sig, frames[0].aux)
arm = (frames[1].sig, frames[1].aux)
samples[point] = (obs, arm)

base_obs = samples[(0, 0, 0)][0]
base_arm = samples[(0, 0, 0)][1]
row_obs_sig = (
(samples[(1, 0, 0)][0][0] - base_obs[0]) % MOD,
(samples[(0, 1, 0)][0][0] - base_obs[0]) % MOD,
(samples[(0, 0, 1)][0][0] - base_obs[0]) % MOD,
)
row_obs_aux = (
(samples[(1, 0, 0)][0][1] - base_obs[1]) % MOD,
(samples[(0, 1, 0)][0][1] - base_obs[1]) % MOD,
(samples[(0, 0, 1)][0][1] - base_obs[1]) % MOD,
)
row_arm_sig = (
(samples[(1, 0, 0)][1][0] - base_arm[0]) % MOD,
(samples[(0, 1, 0)][1][0] - base_arm[0]) % MOD,
(samples[(0, 0, 1)][1][0] - base_arm[0]) % MOD,
)
row_arm_aux = (
(samples[(1, 0, 0)][1][1] - base_arm[1]) % MOD,
(samples[(0, 1, 0)][1][1] - base_arm[1]) % MOD,
(samples[(0, 0, 1)][1][1] - base_arm[1]) % MOD,
)
return base_obs, row_obs_sig, row_obs_aux, base_arm, row_arm_sig,
row_arm_aux

def words_from_nonce(nonce: str) -> tuple[int, int, int]:

return tuple(int(nonce[i : i + 4], 16) % MOD for i in range(0, 12, 4))

def crc16_modbus(data: bytes) -> int:
crc = 0xFFFF
for byte in data:
crc ^= byte
for _ in range(8):
if crc & 1:
crc = (crc >> 1) ^ 0xA001
else:
crc >>= 1
return crc & 0xFFFF

def crc16_ccitt_false(data: bytes) -> int:
crc = 0xFFFF
for byte in data:
crc ^= byte << 8
for _ in range(8):
if crc & 0x8000:
crc = ((crc << 1) ^ 0x1021) & 0xFFFF
else:
crc = (crc << 1) & 0xFFFF
return crc

def crc16_x25(data: bytes) -> int:
crc = 0xFFFF
for byte in data:
crc ^= byte
for _ in range(8):
if crc & 1:
crc = (crc >> 1) ^ 0x8408
else:
crc >>= 1
return (~crc) & 0xFFFF

def fletcher16(data: bytes) -> int:
s1 = 0
s2 = 0
for byte in data:
s1 = (s1 + byte) % 255
s2 = (s2 + s1) % 255
return ((s2 << 8) | s1) & 0xFFFF

def internet_checksum(data: bytes) -> int:
if len(data) & 1:
data += b"\x00"
total = 0

for i in range(0, len(data), 2):
total += (data[i] << 8) | data[i + 1]
total = (total & 0xFFFF) + (total >> 16)
return (~total) & 0xFFFF

def checksum_sum16_be(data: bytes) -> int:
if len(data) & 1:
data += b"\x00"
total = 0
for i in range(0, len(data), 2):
total = (total + ((data[i] << 8) | data[i + 1])) & 0xFFFF
return total

def checksum_sum16_le(data: bytes) -> int:
if len(data) & 1:
data += b"\x00"
total = 0
for i in range(0, len(data), 2):
total = (total + (data[i] | (data[i + 1] << 8))) & 0xFFFF
return total

def build_checksum_candidates(chal: Frame, nonce: str) -> list[tuple[str,
int]]:
nonce_bytes = bytes.fromhex(nonce)
ttl_match = re.search(r"ttl=(\d+)", chal.rest)
ttl = int(ttl_match.group(1)) if ttl_match else 0

payloads: list[tuple[str, bytes]] = [
("chal_ascii_full", chal.raw.encode()),
("chal_ascii_tail", f"sig={chal.sig} aux={chal.aux} tag=CHAL
{chal.rest}".encode()),
("chal_ascii_nonce", f"{chal.sig}:{chal.aux}:{nonce}:{ttl}".encode()),
(
"chal_bin_be",
chal.sig.to_bytes(2, "big")
+ chal.aux.to_bytes(2, "big")
+ nonce_bytes
+ ttl.to_bytes(2, "big"),
),
(
"chal_bin_le",
chal.sig.to_bytes(2, "little")
+ chal.aux.to_bytes(2, "little")
+ nonce_bytes
+ ttl.to_bytes(2, "little"),
),
(

"chal_bin_with_lane",
bytes([chal.lane & 0xFF])
+ chal.sig.to_bytes(2, "big")
+ chal.aux.to_bytes(2, "big")
+ nonce_bytes
+ ttl.to_bytes(2, "big"),
),
(
"chal_words_be",
b"".join(word.to_bytes(2, "big") for word in
words_from_nonce(nonce))
+ chal.sig.to_bytes(2, "big")
+ chal.aux.to_bytes(2, "big"),
),
]
algos: list[tuple[str, callable]] = [
("crc16_modbus", crc16_modbus),
("crc16_x25", crc16_x25),
("crc16_ccitt", crc16_ccitt_false),
("crc16_xmodem", lambda data: binascii.crc_hqx(data, 0)),
("fletcher16", fletcher16),
("internet", internet_checksum),
("sum16_be", checksum_sum16_be),
("sum16_le", checksum_sum16_le),
]

out: list[tuple[str, int]] = []
seen: set[tuple[str, int]] = set()
for payload_name, payload in payloads:
for algo_name, algo in algos:
for mod_name, value in [
("raw16", algo(payload) & 0xFFFF),
("mod65521", algo(payload) % MOD),
]:
label = f"{algo_name}:{payload_name}:{mod_name}"
key = (label, value)
if key not in seen:
seen.add(key)
out.append((label, value))
return out

def challenge_for_commit(
mb: MB9,
commit_point: tuple[int, int, int],
timeout: float = 0.8,
) -> tuple[str, Frame, Frame, str]:
raw = mb.batch(

[
"RESET",
f"ENQ MIX {commit_point[0]} {commit_point[1]} {commit_point[2]}",
"ARM",
"COMMIT",
"POLL 16",
],
timeout=timeout,
)
frames = parse_frames(raw)
nonce = NONCE_RE.search(raw).group(1)
return nonce, frames[0], frames[1], raw

def cmd_try_chal_checksums(args: argparse.Namespace) -> None:
mb = MB9(args.port, timeout=args.timeout)
_, _, _, base_arm, row_arm_sig, row_arm_aux = learn_reset_maps(mb)
commit_point = solve_affine_line(base_arm[0], base_arm[1], row_arm_sig,
row_arm_aux, args.commit_c)
nonce, obs, chal, raw = challenge_for_commit(mb, commit_point,
timeout=args.timeout)
candidates = build_checksum_candidates(chal, nonce)
if args.limit is not None:
candidates = candidates[: args.limit]

print(f"sid={mb.sid} commit_c={args.commit_c} nonce={nonce}")
print(f"commit_point={commit_point} commit_obs={(obs.sig, obs.aux)}")
print(f"chal_line={chal.raw}")
print(f"candidate_count={len(candidates)}")

checked = 0
while checked < len(candidates):
batch = candidates[checked : checked + args.batch]
prove_lines = [f"PROVE {chal.sig} {chal.aux} {value}" for _, value in
batch]
raw = mb.batch(
[
"RESET",
f"ENQ MIX {commit_point[0]} {commit_point[1]}
{commit_point[2]}",
"ARM",
"COMMIT",
*prove_lines,
],
timeout=max(args.timeout, 0.12),
)

for idx, (label, value) in enumerate(batch):

print(f"try[{checked + idx}] label={label} p3={value}")

if "SUCTF{" in raw or "OK cmd=PROVE" in raw:
print(f"hit_batch_start={checked}")
print(raw, end="" if raw.endswith("\n") else "\n")
mb.close()
return

if "E_LIMIT" in raw or "cmd_budget_exhausted" in raw:
print(f"budget_exhausted after={checked}")
break

checked += len(batch)

mb.close()
print(f"no_hit checked={checked}")

def cmd_bruteforce_chal_checksum(args: argparse.Namespace) -> None:
checked = 0
cur = args.start % MOD
stop = MOD if args.stop is None else min(args.stop, MOD)
chunk = max(1, args.chunk)
session_id = 0

while cur < stop:
session_id += 1
mb = MB9(args.port, timeout=args.timeout)
_, _, _, base_arm, row_arm_sig, row_arm_aux = learn_reset_maps(mb)
commit_point = solve_affine_line(base_arm[0], base_arm[1],
row_arm_sig, row_arm_aux, args.commit_c)
nonce, obs, chal, _ = challenge_for_commit(mb, commit_point,
timeout=args.timeout)
print(
f"session={session_id} sid={mb.sid} nonce={nonce} commit_c=
{args.commit_c} "
f"commit_obs={(obs.sig, obs.aux)} chal={(chal.sig, chal.aux)}"
)

session_cmds = 0
while cur < stop:
vals = list(range(cur, min(stop, cur + chunk)))
raw = mb.batch(
[
"RESET",
f"ENQ MIX {commit_point[0]} {commit_point[1]}
{commit_point[2]}",
"ARM",

"COMMIT",
*(f"PROVE {chal.sig} {chal.aux} {value}" for value in
vals),
],
timeout=max(args.timeout, 0.12),
)

if "SUCTF{" in raw or "OK cmd=PROVE" in raw:
print(f"hit_session={session_id} start_p3={cur}")
print(raw, end="" if raw.endswith("\n") else "\n")
mb.close()
return

if "E_LIMIT" in raw or "cmd_budget_exhausted" in raw:
print(f"budget_exhausted session={session_id} checked=
{checked}")
break

checked += len(vals)
session_cmds += 4 + len(vals)
cur += len(vals)
if args.progress and checked % args.progress == 0:
print(
f"progress checked={checked} next_p3={cur} session=
{session_id} "
f"session_cmds={session_cmds}"
)

mb.close()

print(f"no_hit checked={checked} searched=[{args.start},{stop}) commit_c=
{args.commit_c}")

def active_zero_for_commit(
mb: MB9,
commit_point: tuple[int, int, int],
timeout: float = 0.8,
) -> tuple[Frame, Frame]:
challenge_for_commit(mb, commit_point, timeout=timeout)
raw = mb.batch(["ENQ MIX 0 0 0", "ARM", "COMMIT", "POLL 16"],
timeout=timeout)
frames = parse_frames(raw)
return frames[0], frames[1]

def line_step(
base_sig: int,
base_aux: int,

row_sig: tuple[int, int, int],
row_aux: tuple[int, int, int],
) -> tuple[tuple[int, int, int], tuple[int, int, int]]:
x0 = solve_affine_line(base_sig, base_aux, row_sig, row_aux, 0)
x1 = solve_affine_line(base_sig, base_aux, row_sig, row_aux, 1)
return x0, ((x1[0] - x0[0]) % MOD, (x1[1] - x0[1]) % MOD, (x1[2] - x0[2])
% MOD)

def map_eval(
base: tuple[int, int],
row_sig: tuple[int, int, int],
row_aux: tuple[int, int, int],
point: tuple[int, int, int],
) -> tuple[int, int]:
return (
(base[0] + row_sig[0] * point[0] + row_sig[1] * point[1] + row_sig[2]
* point[2]) % MOD,
(base[1] + row_aux[0] * point[0] + row_aux[1] * point[1] + row_aux[2]
* point[2]) % MOD,
)

def derive_state(commit_c: int, formula: str, ctx: dict[str, int]) -> int:
if formula == "c":
return commit_c % MOD
if formula == "-c":
return (-commit_c) % MOD
if formula == "c+1":
return (commit_c + 1) % MOD
if formula == "0":
return 0
if formula == "1":
return 1
if formula == "chal_sig":
return ctx["chal_sig"]
if formula == "chal_aux":
return ctx["chal_aux"]
if formula == "obs_sig":
return ctx["obs_sig"]
if formula == "obs_aux":
return ctx["obs_aux"]
if formula == "nonce0":
return ctx["nonce0"]
if formula == "nonce1":
return ctx["nonce1"]
if formula == "nonce2":
return ctx["nonce2"]
if formula == "nonce_sum":

return (ctx["nonce0"] + ctx["nonce1"] + ctx["nonce2"]) % MOD
if formula == "c+chal_sig":
return (commit_c + ctx["chal_sig"]) % MOD
if formula == "c+chal_aux":
return (commit_c + ctx["chal_aux"]) % MOD
if formula == "c+obs_sig":
return (commit_c + ctx["obs_sig"]) % MOD
if formula == "c+obs_aux":
return (commit_c + ctx["obs_aux"]) % MOD
if formula == "c+nonce0":
return (commit_c + ctx["nonce0"]) % MOD
if formula == "c+nonce1":
return (commit_c + ctx["nonce1"]) % MOD
if formula == "c+nonce2":
return (commit_c + ctx["nonce2"]) % MOD
raise ValueError(f"unsupported formula: {formula}")

def cmd_measure(args: argparse.Namespace) -> None:
mb = MB9(args.port, timeout=args.timeout)
base_obs, row_obs_sig, row_obs_aux, base_arm, row_arm_sig, row_arm_aux =
learn_reset_maps(mb)
print(f"sid={mb.sid}")
print(f"reset_obs_base={base_obs}")
print(f"reset_obs_rows={row_obs_sig} / {row_obs_aux}")
print(f"reset_arm_base={base_arm}")
print(f"reset_arm_rows={row_arm_sig} / {row_arm_aux}")
for c in args.commit_cs:
commit_point = solve_affine_line(base_arm[0], base_arm[1],
row_arm_sig, row_arm_aux, c)
nonce, obs, chal, _ = challenge_for_commit(mb, commit_point,
timeout=args.timeout)
active_obs, active_arm = active_zero_for_commit(mb, commit_point,
timeout=args.timeout)
print(
f"c={c} commit_point={commit_point} nonce={nonce} "
f"commit_obs={(obs.sig, obs.aux)} chal={(chal.sig, chal.aux)} "
f"active_obs0={(active_obs.sig, active_obs.aux)} active_arm0=
{(active_arm.sig, active_arm.aux)}"
)
mb.close()

def cmd_try_formulas(args: argparse.Namespace) -> None:
mb = MB9(args.port, timeout=args.timeout)
_, row_obs_sig, row_obs_aux, base_arm, row_arm_sig, row_arm_aux =
learn_reset_maps(mb)
commit_point = solve_affine_line(base_arm[0], base_arm[1], row_arm_sig,
row_arm_aux, args.commit_c)

nonce, obs, chal, _ = challenge_for_commit(mb, commit_point,
timeout=args.timeout)
active_obs, active_arm = active_zero_for_commit(mb, commit_point,
timeout=args.timeout)
ctx = {
"chal_sig": chal.sig,
"chal_aux": chal.aux,
"obs_sig": obs.sig,
"obs_aux": obs.aux,
"nonce0": words_from_nonce(nonce)[0],
"nonce1": words_from_nonce(nonce)[1],
"nonce2": words_from_nonce(nonce)[2],
}

candidates: list[tuple[str, tuple[int, int, int]]] = []
for formula in args.formulas:
c = derive_state(args.commit_c, formula, ctx)
point = solve_affine_line(active_arm.sig, active_arm.aux, row_arm_sig,
row_arm_aux, c)
candidates.append((formula, point))

lines = [
"RESET",
f"ENQ MIX {commit_point[0]} {commit_point[1]} {commit_point[2]}",
"ARM",
"COMMIT",
"POLL 16",
]
lines.extend(f"PROVE {point[0]} {point[1]} {point[2]}" for _, point in
candidates)
raw = mb.batch(lines, timeout=max(args.timeout, 0.9))

print(f"sid={mb.sid} commit_c={args.commit_c} nonce={nonce}")
print(f"commit_obs={(obs.sig, obs.aux)} chal={(chal.sig, chal.aux)}")
print(f"active_obs0={(active_obs.sig, active_obs.aux)} active_arm0=
{(active_arm.sig, active_arm.aux)}")
print(f"nonce_words={words_from_nonce(nonce)}")
for label, point in candidates:
print(f"formula={label} point={point}")
print(raw, end="" if raw.endswith("\n") else "\n")
mb.close()

def cmd_bruteforce_active_line(args: argparse.Namespace) -> None:
checked = 0
cur = args.start % MOD
stop = MOD if args.stop is None else min(args.stop, MOD)
chunk = max(1, args.chunk)

session_id = 0

while cur < stop:
session_id += 1
mb = MB9(args.port, timeout=args.timeout)
_, _, _, base_arm, row_arm_sig, row_arm_aux = learn_reset_maps(mb)
commit_point = solve_affine_line(base_arm[0], base_arm[1],
row_arm_sig, row_arm_aux, args.commit_c)
nonce, obs, chal, _ = challenge_for_commit(mb, commit_point,
timeout=args.timeout)
_, active_arm = active_zero_for_commit(mb, commit_point,
timeout=args.timeout)
line_x0, line_w = line_step(active_arm.sig, active_arm.aux,
row_arm_sig, row_arm_aux)

print(
f"session={session_id} sid={mb.sid} nonce={nonce} commit_c=
{args.commit_c} "
f"commit_obs={(obs.sig, obs.aux)} chal={(chal.sig, chal.aux)} "
f"active_arm0={(active_arm.sig, active_arm.aux)} line_x0={line_x0}
line_w={line_w}"
)

session_cmds = 0
while cur < stop:
pts: list[tuple[int, int, int]] = []
prove_lines: list[str] = []
end = min(stop, cur + chunk)
for t in range(cur, end):
point = (
(line_x0[0] + line_w[0] * t) % MOD,
(line_x0[1] + line_w[1] * t) % MOD,
(line_x0[2] + line_w[2] * t) % MOD,
)
pts.append(point)
prove_lines.append(f"PROVE {point[0]} {point[1]} {point[2]}")

raw = mb.batch(
[
"RESET",
f"ENQ MIX {commit_point[0]} {commit_point[1]}
{commit_point[2]}",
"ARM",
"COMMIT",
*prove_lines,
],
timeout=max(args.timeout, 1.0),

)

if "SUCTF{" in raw or "OK cmd=PROVE" in raw:
print(f"hit_session={session_id} start_t={cur}")
for idx, line in enumerate(raw.splitlines()):
if "SUCTF{" in line or line.startswith("OK cmd=PROVE"):
point = pts[idx] if idx < len(pts) else None
print(f"hit_t={cur + idx} point={point}")
print(raw, end="" if raw.endswith("\n") else "\n")
mb.close()
return

if "E_LIMIT" in raw or "cmd_budget_exhausted" in raw:
print(f"budget_exhausted session={session_id} checked=
{checked}")
break

checked += len(pts)
session_cmds += 4 + len(pts)
cur = end
if args.progress and checked % args.progress == 0:
print(f"progress checked={checked} next_t={cur} session=
{session_id} session_cmds={session_cmds}")

mb.close()

print(f"no_hit checked={checked} searched=[{args.start},{stop}) commit_c=
{args.commit_c}")

def solve_target_family_point(
base_pair: tuple[int, int],
row_sig: tuple[int, int, int],
row_aux: tuple[int, int, int],
target_pair: tuple[int, int],
z: int,
) -> tuple[int, int, int]:
det = (row_sig[0] * row_aux[1] - row_sig[1] * row_aux[0]) % MOD
rhs_sig = (target_pair[0] - base_pair[0] - row_sig[2] * z) % MOD
rhs_aux = (target_pair[1] - base_pair[1] - row_aux[2] * z) % MOD
inv = pow(det, -1, MOD)
a = (rhs_sig * row_aux[1] - row_sig[1] * rhs_aux) % MOD * inv % MOD
b = (row_sig[0] * rhs_aux - rhs_sig * row_aux[0]) % MOD * inv % MOD
return a, b, z % MOD

def cmd_bruteforce_family(args: argparse.Namespace) -> None:
checked = 0
cur = args.start % MOD

stop = MOD if args.stop is None else min(args.stop, MOD)
chunk = max(1, args.chunk)
session_id = 0

while cur < stop:
session_id += 1
mb = MB9(args.port, timeout=args.timeout)
reset_obs_base, row_obs_sig, row_obs_aux, base_arm, row_arm_sig,
row_arm_aux = learn_reset_maps(mb)
commit_point = solve_affine_line(base_arm[0], base_arm[1],
row_arm_sig, row_arm_aux, args.commit_c)
nonce, obs, chal, _ = challenge_for_commit(mb, commit_point,
timeout=args.timeout)
active_obs, active_arm = active_zero_for_commit(mb, commit_point,
timeout=args.timeout)

family_rows = {
"arm": ((active_arm.sig, active_arm.aux), row_arm_sig,
row_arm_aux),
"obs": ((active_obs.sig, active_obs.aux), row_obs_sig,
row_obs_aux),
}
targets = {
"zero": (0, 0),
"chal": (chal.sig, chal.aux),
"commit_obs": (obs.sig, obs.aux),
"active_obs": (active_obs.sig, active_obs.aux),
"active_arm": (active_arm.sig, active_arm.aux),
"reset_obs": reset_obs_base,
"reset_arm": base_arm,
}
base_pair, row_sig, row_aux = family_rows[args.family]
target_pair = targets[args.target]
line_x0 = solve_target_family_point(base_pair, row_sig, row_aux,
target_pair, 0)
line_x1 = solve_target_family_point(base_pair, row_sig, row_aux,
target_pair, 1)
line_w = (
(line_x1[0] - line_x0[0]) % MOD,
(line_x1[1] - line_x0[1]) % MOD,
(line_x1[2] - line_x0[2]) % MOD,
)

print(
f"session={session_id} sid={mb.sid} nonce={nonce} commit_c=
{args.commit_c} "

f"family={args.family} target={args.target} target_pair=
{target_pair} "
f"line_x0={line_x0} line_w={line_w}"
)

session_cmds = 0
while cur < stop:
pts: list[tuple[int, int, int]] = []
prove_lines: list[str] = []
end = min(stop, cur + chunk)
for t in range(cur, end):
point = (
(line_x0[0] + line_w[0] * t) % MOD,
(line_x0[1] + line_w[1] * t) % MOD,
(line_x0[2] + line_w[2] * t) % MOD,
)
pts.append(point)
prove_lines.append(f"PROVE {point[0]} {point[1]} {point[2]}")

raw = mb.batch(
[
"RESET",
f"ENQ MIX {commit_point[0]} {commit_point[1]}
{commit_point[2]}",
"ARM",
"COMMIT",
*prove_lines,
],
timeout=max(args.timeout, 1.0),
)

if "SUCTF{" in raw or "OK cmd=PROVE" in raw:
print(f"hit_session={session_id} start_t={cur}")
print(raw, end="" if raw.endswith("\n") else "\n")
mb.close()
return

if "E_LIMIT" in raw or "cmd_budget_exhausted" in raw:
print(f"budget_exhausted session={session_id} checked=
{checked}")
break

checked += len(pts)
session_cmds += 4 + len(pts)
cur = end
if args.progress and checked % args.progress == 0:

print(f"progress checked={checked} next_t={cur} session=
{session_id} session_cmds={session_cmds}")

mb.close()

print(
f"no_hit checked={checked} searched=[{args.start},{stop}) commit_c=
{args.commit_c} "
f"family={args.family} target={args.target}"
)

def _current_line_x0(
mb: MB9,
commit_point: tuple[int, int, int],
row_arm_sig: tuple[int, int, int],
row_arm_aux: tuple[int, int, int],
depth: int,
timeout: float,
) -> tuple[int, int, int]:
prefix = [
"RESET",
f"ENQ MIX {commit_point[0]} {commit_point[1]} {commit_point[2]}",
"ARM",
"COMMIT",
"POLL 16",
]
for _ in range(depth):
prefix.extend(["ARM", "COMMIT", "POLL 4"])
raw = mb.batch([*prefix, "ENQ MIX 0 0 0", "ARM", "COMMIT", "POLL 8"],
timeout=max(timeout, 1.4))
frames = parse_frames(raw)
arm = frames[-1]
return solve_affine_line(arm.sig, arm.aux, row_arm_sig, row_arm_aux, 0)

def _next_line_x0_from_valid_mix(
mb: MB9,
commit_point: tuple[int, int, int],
row_arm_sig: tuple[int, int, int],
row_arm_aux: tuple[int, int, int],
line_w: tuple[int, int, int],
depth: int,
mix_t: int,
timeout: float,
) -> tuple[int, int, int]:
cur_x0 = _current_line_x0(mb, commit_point, row_arm_sig, row_arm_aux,
depth, timeout)
point = (

(cur_x0[0] + line_w[0] * mix_t) % MOD,
(cur_x0[1] + line_w[1] * mix_t) % MOD,
(cur_x0[2] + line_w[2] * mix_t) % MOD,
)
prefix = [
"RESET",
f"ENQ MIX {commit_point[0]} {commit_point[1]} {commit_point[2]}",
"ARM",
"COMMIT",
"POLL 16",
]
for _ in range(depth):
prefix.extend(["ARM", "COMMIT", "POLL 4"])
raw = mb.batch(
[
*prefix,
f"ENQ MIX {point[0]} {point[1]} {point[2]}",
"ARM",
"COMMIT",
"POLL 8",
"ENQ MIX 0 0 0",
"ARM",
"COMMIT",
"POLL 8",
],
timeout=max(timeout, 1.8),
)
frames = parse_frames(raw)
arm = frames[-1]
return solve_affine_line(arm.sig, arm.aux, row_arm_sig, row_arm_aux, 0)

def cmd_bruteforce_aligned_active_line(args: argparse.Namespace) -> None:
checked = 0
cur = args.start % MOD
stop = MOD if args.stop is None else min(args.stop, MOD)
chunk = max(1, args.chunk)
session_id = 0

while cur < stop:
session_id += 1
mb = MB9(args.port, timeout=args.timeout)
_, _, _, base_arm, row_arm_sig, row_arm_aux = learn_reset_maps(mb)
reset_x0, reset_w = line_step(base_arm[0], base_arm[1], row_arm_sig,
row_arm_aux)
commit0 = solve_affine_line(base_arm[0], base_arm[1], row_arm_sig,
row_arm_aux, 0)

commit1 = solve_affine_line(base_arm[0], base_arm[1], row_arm_sig,
row_arm_aux, 1)

cur0 = _current_line_x0(mb, commit0, row_arm_sig, row_arm_aux,
args.depth, args.timeout)
cur1 = _current_line_x0(mb, commit1, row_arm_sig, row_arm_aux,
args.depth, args.timeout)
next0 = _next_line_x0_from_valid_mix(
mb, commit0, row_arm_sig, row_arm_aux, reset_w, args.depth, 0,
args.timeout
)
next1 = _next_line_x0_from_valid_mix(
mb, commit1, row_arm_sig, row_arm_aux, reset_w, args.depth, 0,
args.timeout
)
canon_next0 = _current_line_x0(mb, commit0, row_arm_sig, row_arm_aux,
args.depth + 1, args.timeout)

v = ((cur1[0] - cur0[0]) % MOD, (cur1[1] - cur0[1]) % MOD)
u = ((next1[0] - next0[0]) % MOD, (next1[1] - next0[1]) % MOD)
d = ((next0[0] - canon_next0[0]) % MOD, (next0[1] - canon_next0[1]) %
MOD)
det_uv = (u[0] * v[1] - u[1] * v[0]) % MOD
if det_uv == 0:
print(
f"session={session_id} sid={mb.sid} aligned_skip=det0 depth=
{args.depth} "
f"cur={cur0} next0={next0} canon_next0={canon_next0} v={v} u=
{u}"
)
mb.close()
continue

c_star = (-(d[0] * v[1] - d[1] * v[0])) % MOD * pow(det_uv, -1, MOD) %
MOD
commit_star = solve_affine_line(base_arm[0], base_arm[1], row_arm_sig,
row_arm_aux, c_star)
nonce, obs, chal, _ = challenge_for_commit(mb, commit_star,
timeout=args.timeout)
_, active_arm = active_zero_for_commit(mb, commit_star,
timeout=args.timeout)
line_x0, line_w = line_step(active_arm.sig, active_arm.aux,
row_arm_sig, row_arm_aux)

print(
f"session={session_id} sid={mb.sid} depth={args.depth} c_star=
{c_star} nonce={nonce} "

f"commit_obs={(obs.sig, obs.aux)} chal={(chal.sig, chal.aux)} "
f"line_x0={line_x0} line_w={line_w}"
)

session_cmds = 0
while cur < stop:
pts: list[tuple[int, int, int]] = []
prove_lines: list[str] = []
end = min(stop, cur + chunk)
for t in range(cur, end):
point = (
(line_x0[0] + line_w[0] * t) % MOD,
(line_x0[1] + line_w[1] * t) % MOD,
(line_x0[2] + line_w[2] * t) % MOD,
)
pts.append(point)
prove_lines.append(f"PROVE {point[0]} {point[1]} {point[2]}")

raw = mb.batch(
[
"RESET",
f"ENQ MIX {commit_star[0]} {commit_star[1]}
{commit_star[2]}",
"ARM",
"COMMIT",
*prove_lines,
],
timeout=max(args.timeout, 1.0),
)

if "SUCTF{" in raw or "OK cmd=PROVE" in raw:
print(f"hit_session={session_id} start_t={cur}")
print(raw, end="" if raw.endswith("\n") else "\n")
mb.close()
return

if "E_LIMIT" in raw or "cmd_budget_exhausted" in raw:
print(f"budget_exhausted session={session_id} checked=
{checked}")
break

checked += len(pts)
session_cmds += 4 + len(pts)
cur = end
if args.progress and checked % args.progress == 0:
print(f"progress checked={checked} next_t={cur} session=
{session_id} session_cmds={session_cmds}")

mb.close()

print(f"no_hit checked={checked} searched=[{args.start},{stop})
aligned_active_line depth={args.depth}")

def build_parser() -> argparse.ArgumentParser:
parser = argparse.ArgumentParser()
sub = parser.add_subparsers(dest="cmd", required=True)

p = sub.add_parser("measure")
p.add_argument("--port", type=int, default=10011)
p.add_argument("--timeout", type=float, default=0.8)
p.add_argument("--commit-cs", type=int, nargs="+", default=[0, 1, 2, 3])
p.set_defaults(func=cmd_measure)

p = sub.add_parser("try-formulas")
p.add_argument("--port", type=int, default=10011)
p.add_argument("--timeout", type=float, default=0.8)
p.add_argument("--commit-c", type=int, default=12345)
p.add_argument(
"formulas",
nargs="+",
help=(
"Supported: 0 1 c -c c+1 chal_sig chal_aux obs_sig obs_aux "
"nonce0 nonce1 nonce2 nonce_sum c+chal_sig c+chal_aux c+obs_sig
```

$$
c+obs_aux c+nonce0 c+nonce1 c+nonce2"
$$

```python
),
)
p.set_defaults(func=cmd_try_formulas)

p = sub.add_parser("try-chal-checksums")
p.add_argument("--port", type=int, default=10011)
p.add_argument("--timeout", type=float, default=0.8)
p.add_argument("--commit-c", type=int, default=0)
p.add_argument("--batch", type=int, default=7)
p.add_argument("--limit", type=int, default=None)
p.set_defaults(func=cmd_try_chal_checksums)

p = sub.add_parser("bruteforce-chal-checksum")
p.add_argument("--port", type=int, default=10011)
p.add_argument("--timeout", type=float, default=0.8)
p.add_argument("--commit-c", type=int, default=0)
p.add_argument("--start", type=int, default=0)
p.add_argument("--stop", type=int, default=None)
p.add_argument("--chunk", type=int, default=7)
p.add_argument("--progress", type=int, default=700)

p.set_defaults(func=cmd_bruteforce_chal_checksum)

p = sub.add_parser("bruteforce-active-line")
p.add_argument("--port", type=int, default=10011)
p.add_argument("--timeout", type=float, default=0.8)
p.add_argument("--commit-c", type=int, default=0)
p.add_argument("--start", type=int, default=0)
p.add_argument("--stop", type=int, default=None)
p.add_argument("--chunk", type=int, default=7)
p.add_argument("--progress", type=int, default=700)
p.set_defaults(func=cmd_bruteforce_active_line)

p = sub.add_parser("bruteforce-family")
p.add_argument("--port", type=int, default=10011)
p.add_argument("--timeout", type=float, default=0.8)
p.add_argument("--commit-c", type=int, default=0)
p.add_argument("--family", choices=["arm", "obs"], required=True)
p.add_argument(
"--target",
choices=["zero", "chal", "commit_obs", "active_obs", "active_arm",
"reset_obs", "reset_arm"],
required=True,
)
p.add_argument("--start", type=int, default=0)
p.add_argument("--stop", type=int, default=None)
p.add_argument("--chunk", type=int, default=7)
p.add_argument("--progress", type=int, default=700)
p.set_defaults(func=cmd_bruteforce_family)

p = sub.add_parser("bruteforce-aligned-active-line")
p.add_argument("--port", type=int, default=10011)
p.add_argument("--timeout", type=float, default=0.8)
p.add_argument("--depth", type=int, default=0)
p.add_argument("--start", type=int, default=0)
p.add_argument("--stop", type=int, default=None)
p.add_argument("--chunk", type=int, default=7)
p.add_argument("--progress", type=int, default=700)
p.set_defaults(func=cmd_bruteforce_aligned_active_line)

return parser

def main() -> None:
parser = build_parser()
args = parser.parse_args()
args.func(args)

if __name__ == "__main__":

main()
```

直接跑：

```
py -u mb9_search.py bruteforce-chal-checksum --commit-c 0 --timeout 0.12 --
progress 7000
```

这里的思路是：

1. 每个新 TCP session 先学 reset 态下的 ARM 仿射映射。

2. 求出一个能触发 CHAL 的 commit_point 。

3. 固定 p1 = chal.sig 、p2 = chal.aux 。

4. 对 p3 in [0, 65520] 分批爆破。

把批量交互超时压到 0.12s 以后，速度就够用了。

命中过程

实际命中的日志如下：

```
session=1 sid=a54c042470ba6bb6 nonce=621914002e99 commit_c=0 commit_obs=
(14489, 5557) chal=(61405, 29725)
budget_exhausted session=1 checked=2590
session=2 sid=408b5c3172077b47 nonce=175a6f7bf012 commit_c=0 commit_obs=
(42840, 44217) chal=(60056, 41938)
hit_session=2 start_p3=3458
OK cmd=RESET tick=0 phase=0 qlen=0 backlog=0
QOK qid=1 opcode=MIX argc=3 qlen=1
QOK qid=2 opcode=ARM argc=0 qlen=2
COK cid=1 exec=2 produced=2 qlen=0 backlog=2 tick=2 phase=0
ERR code=E_PROVE msg=bad_proof
OK cmd=PROVE status=PASS flag=SUCTF{mb9_file_only_flag_runtime_hardened}
ERR code=E_STATE msg=no_active_challenge
ERR code=E_STATE msg=no_active_challenge
ERR code=E_STATE msg=no_active_challenge
ERR code=E_STATE msg=no_active_challenge
ERR code=E_STATE msg=no_active_challenge
```

这里 start_p3=3458 ，而回包里是先错一次再成功一次，所以真实命中的值是：

```
p3 = 3459
```

也就是这一组参数成功：

```
PROVE 60056 41938 3459
```

## 方法总结
- 核心技巧：有限域仿射协议学习
- 识别信号：协议状态机可 reset，命令输出对输入呈线性/仿射关系。
- 复用要点：采样零点和基向量恢复矩阵，求解触发条件，再重复学习后续阶段的 affine map。

