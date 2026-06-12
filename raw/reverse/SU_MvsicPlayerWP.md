# SU_MvsicPlayer

## 题目简述
题目是 Electron 音乐播放器逆向。题目明确 flag 提交格式为 `SUCTF{wav文件的md5值}`。关键文件包括 `app.asar`、native `.node` 模块和 `.su_mv_enc` 数据；目标从恢复完整 `.su_mv` 改为提交原始 WAV 的 MD5，因此重点是理解前端封装、native bridge 和 VM 加密流程，恢复 WAV payload。

## 解题过程
1. 题目分析

附件是一个 Electron 程序，目录里最关键的几个文件是：
- win-unpacked/resources/app.asar
- app_asar_extracted/native/build/Release/vm_encryptor.node
- ddd.su_mv_enc

题目原本要求恢复 .su_mv ，后来改成只需要提交原始 wav 的 md5 。这意味着本题的核心目标可
以简化成一句话：

把 ddd.su_mv_enc 对应的原始 WAV payload 恢复出来，然后计算 md5。

2. 先解包 Electron

先把前端逻辑拆出来：

```
npx asar extract win-unpacked/resources/app.asar app_asar_extracted
```

解包后重点看 3 个文件：

- src/common/sumv-browser.js
- src/renderer/app.js
- src/main/native-bridge.js

前端逻辑并不复杂：

1. 选择一个 .su_mv 文件

2. 用 SUMV.parseSuMv() 解析出 payload

3. 用浏览器音频组件播放 payload

4. 播放结束或关闭窗口时，把 payload 交给 vmEncrypt() ，写成 *_enc

所以 ddd.su_mv_enc 并不是“整个 .su_mv 文件的加密结果”，而是 .su_mv 解析出的音频
payload 的加密结果。

这一点非常关键。

3. .su_mv 文件格式分析

sumv-browser.js 里直接给出了 .su_mv 的解析逻辑，可以整理成下面的格式：

- 文件头 SUMV
- offset 0x04 ：version
- offset 0x06 ：formatCode
- offset 0x08 ：解压后长度 u32le
- offset 0x0C ：压缩数据长度 u32le
- offset 0x10 起：压缩数据

之后会经过两步处理：

1. 自定义解压 _5bb006

2. 一个 RC4 风格的异或流，还原 key 为 SUMUSICPLAYER

也就是说，.su_mv -> payload 这一步其实已经是明牌了，真正的难点不在容器，而在 payload
被怎样加密成了 ddd.su_mv_enc 。

4. 先不要被 JS 里的 placeholder 误导

native-bridge.js 里有一个 placeholderVmEncrypt() ，逻辑大概是：

- 开头加 SVE4

- 维护一个状态字节

- 每个字节先异或，再做循环左移

如果只看这里，很容易以为题目就是把 SVE4 逆掉。

但这只是障眼法。

我做过黑盒验证：

- 对随机字节串调用原生 vm_encryptor.node

- 再和 JS 里的 placeholderVmEncrypt() 比较

结论是：

- 非 WAV 数据：原生输出和 placeholder 完全一致

- 合法 WAV 数据：原生输出和 placeholder 完全不同

所以题目真正的坑点是：

vm_encryptor.node 对 WAV 有单独分支。`

5. IDA 里定位真实入口

用 IDA 打开 vm_encryptor.node 后，先看导出：
- node_api_module_get_api_version_v1
- napi_register_module_v1

napi_register_module_v1 的逻辑很简单，它只注册了一个属性，名字就是 vmEncrypt ，
对应的回调函数是：

- sub_180007380

这个回调就是整个 native 加密的真实入口。

6. sub_180007380 的关键分支

sub_180007380 做了三件事：

1. 检查参数是不是 Buffer

2. 读取 Buffer 指针和长度

3. 判断数据是不是合法 WAV

它对 WAV 的判断条件非常严格：

- 必须是 RIFF/WAVE

- 必须有 fmt chunk

- 必须有 data chunk

，也就是 PCM• audioFormat == 1
- bitsPerSample == 16

在 1..8 • channels
- blockAlign == 2 * channels

如果不满足这些条件，走的是：

- sub_180001150

这条路就是 JS placeholder 那套 SVE4 + xor + rol8 。

如果满足这些条件，走的是：

- sub_180001380

这条才是真正的加密逻辑。

7. 为什么题目文件一定要走 WAV 分支

这一点可以动态验证。

对一个标准 PCM WAV 调用 vmEncrypt() ，得到的结果有两个明显特征：

- 和 placeholder 输出完全不同

- 去掉前面的 SVE4 后，长度会按 0x40 对齐

例如：

- 输入 46 字节 WAV，输出内层长度 64

- 输入 108 字节 WAV，输出内层长度 128

- 输入 244 字节 WAV，输出内层长度 256

这说明它不是简单逐字节异或，而是进入了一个按 64-byte 处理的专门分支。

而 ddd.su_mv_enc 的长度正好也符合这个分支的特征，所以不能再按 placeholder 去逆。

8. sub_180001380 的整体流水线

sub_180001380 的结构可以概括成：

1. 申请两个大小为 n + 64 的缓冲区

2. 调用 sub_180002E00 生成一段固定 VM bytecode

3. 调用 sub_180001D90 解析 bytecode，修复跳转目标

4. 调用 sub_1800023E0 解释执行这段 bytecode

5. 从第二个工作缓冲区取结果，前面加上 SVE4

其中最重要的结论是：

sub_180002E00 生成的 bytecode 是固定的，不依赖输入内容。`

我直接从模块里把它抠出来以后，得到：

- bytecode 总长度：19493
- 指令总数：9199

9. VM 指令集整理

sub_1800023E0 其实就是一个非常普通的栈式虚拟机，核心指令如下：
- 0 ：结束
- 1/2/3/4 ：push 立即数
- 5 ：push 寄存器
- 6 ：pop -> 寄存器
- 7 ：add
- 8 ：sub
- 9 ：mul
- 10 ：div
- 11 ：xor
- 12 ：and
- 13 ：or
- 14 ：==
- 15 ：<
- 16 ：jmp
- 17 ：条件跳转，flag 为真
- 18 ：条件跳转，flag 为假
- 19/20 ：读写 u8
- 23/24 ：读写 u32
- 25/26 ：读写 u64
- 27 ：shl
- 28 ：shr
- 29 ：dup
- 30 ：swap
- 31 ：pop 丢弃

这说明所谓“native 加密”本质上不是黑箱汇编，而是一套固定 VM 程序。

10. VM 在做什么

结合 bytecode 和动态行为，可以得到两个关键观察：

1. 它确实是 64-byte 分块处理

2. 不是完全独立块，而是有前向链式依赖

验证方式很简单：

- 只改最后一个采样点，只会明显影响最后一个 64-byte 块

- 改最前面的采样点，会影响当前块以及后续块

所以它更像是：

- 先把 PCM WAV payload 按 64-byte 对齐

- 再做一套自定义的块变换

- 并且块之间存在依赖

这也是为什么直接把 SVE4 逆掉会得到垃圾数据。

做到这里，题目实际上已经被拆成了两层：

1. .su_mv 容器层 这一层已经完全公开，sumv-browser.js 里直接给了解析逻辑。

2. `WAV -> ddd.su_mv_enc` 的 native 加密层。这一层的本质是固定 bytecode、固定 VM、固定 64-byte 块流程。

因此：

1. 把 sub_180002E00 生成的 bytecode 抽出来

2. 按 sub_1800023E0 自己实现解释器

3. 在解释器层面逆这套块变换

4. 直接恢复原始 WAV

5. 计算 md5(wav)

### Exp:

```python
import collections
import hashlib
import struct
import subprocess
import sys
import wave
from pathlib import Path

MASK32 = 0xFFFFFFFF

C1 = 0x62616F7A
C2 = 0x6F6E6777

C3 = 0x696E6221

INIT_A = 0xE3A8C8D6
INIT_SUM = 0x70336364
DELTA_SUM = 0x70336364

RC4_KEY = b"SUMUSICPLAYER"
NATIVE_MODULE =
Path("app_asar_extracted/native/build/Release/vm_encryptor.node")

def rol32(x: int, r: int) -> int:
return ((x << r) | (x >> (32 - r))) & MASK32

def ror32(x: int, r: int) -> int:
return ((x >> r) | (x << (32 - r))) & MASK32

def rc4_crypt(data: bytes, key: bytes = RC4_KEY) -> bytes:
s = list(range(256))
j = 0
for i in range(256):
j = (j + s[i] + key[i % len(key)]) & 0xFF
s[i], s[j] = s[j], s[i]

out = bytearray(len(data))
i = 0
j = 0
for n, b in enumerate(data):
i = (i + 1) & 0xFF
j = (j + s[i]) & 0xFF
s[i], s[j] = s[j], s[i]
out[n] = b ^ s[(s[i] + s[j]) & 0xFF]
return bytes(out)

def schedule(k_words: list[int], a_word: int) -> list[int]:
k = k_words[:]
k[0] = (k[0] + rol32(k[1] ^ a_word, 3)) & MASK32
k[1] = (k[1] + rol32(k[2] ^ k[0], 5)) & MASK32
k[2] = (k[2] + rol32(k[3] ^ k[1], 7)) & MASK32
k[3] = (k[3] + rol32(k[4] ^ k[2], 11)) & MASK32
k[4] = (k[4] + rol32(k[5] ^ k[3], 13)) & MASK32
k[5] = (k[5] + rol32(k[6] ^ k[4], 17)) & MASK32
k[6] = (k[6] + rol32(k[7] ^ k[5], 19)) & MASK32
k[7] = (k[7] + rol32(k[0] ^ k[6], 23)) & MASK32
return k

def derive_subkeys(k_words: list[int], a_word: int) -> tuple[list[int],
list[int], list[int]]:

ka = [
k_words[0] ^ k_words[2] ^ a_word,
k_words[1] ^ k_words[3] ^ ((a_word + C1) & MASK32),
k_words[4] ^ k_words[6] ^ ((a_word + C2) & MASK32),
k_words[5] ^ k_words[7] ^ ((a_word + C3) & MASK32),
]
kb = [
(k_words[0] + k_words[4]) & MASK32,
(k_words[1] + k_words[5]) & MASK32,
(k_words[2] + k_words[6]) & MASK32,
(k_words[3] + k_words[7]) & MASK32,
]
kc = [
k_words[0] ^ k_words[5],
k_words[1] ^ k_words[6],
k_words[2] ^ k_words[7],
k_words[3] ^ k_words[4],
]
return ka, kb, kc

def g_func(t_words: list[int], kb: list[int], kc: list[int], sum_word: int) ->
list[int]:
keys = kb + kc
out = []
for i in range(8):
a = ((((t_words[i] << 4) & MASK32) ^ (t_words[i] >> 5)) + t_words[(i +
1) & 7]) & MASK32
a ^= (sum_word + keys[i]) & MASK32

rot = ((i + 1) & 7) or 8
shr = (i + 1) & 7
b = rol32(t_words[(i + 3) & 7], rot) ^ (sum_word >> shr)
out.append((a + b) & MASK32)
return out

def inv_speck_pairs(t_words: list[int], ka: list[int]) -> list[int]:
out = []
for lane in range(4):
x1 = t_words[2 * lane]
y1 = t_words[2 * lane + 1]
y0 = ror32(y1 ^ x1, 3)
x0 = rol32(((x1 ^ ka[lane]) - y0) & MASK32, 8)
out.extend([x0, y0])
return out

def decrypt_block(cipher_words: list[int], h_words: list[int]) -> list[int]:
left = cipher_words[:8]

right = cipher_words[8:]

round_info = []
k_words = h_words[:]
a_word = INIT_A
sum_word = INIT_SUM

for rnd in range(4):
k_words = schedule(k_words, a_word)
ka, kb, kc = derive_subkeys(k_words, a_word)
round_info.append((ka, kb, kc, sum_word))
a_word = (a_word + 0x70336365 + rnd) & MASK32
sum_word = (sum_word + DELTA_SUM) & MASK32

for ka, kb, kc, sum_word in reversed(round_info):
old_right = inv_speck_pairs(left, ka)
tmp = g_func(left, kb, kc, sum_word)
old_left = [(right[i] ^ tmp[i]) & MASK32 for i in range(8)]
left, right = old_left, old_right

return left + right

def decrypt_vm_encryptor_output(enc_path: Path, wav_out_path: Path) -> bytes:
blob = enc_path.read_bytes()
if blob[:4] != b"SVE4":
raise ValueError("unexpected header")

inner = blob[4:]
if len(inner) % 64 != 0:
raise ValueError("ciphertext length is not 64-byte aligned")

h_words = [
0x00010203,
0x04050607,
0x08090A0B,
0x0C0D0E0F,
0x10111213,
0x14151617,
0x18191A1B,
0x1C1D1E1F,
]

out = bytearray()
for block_off in range(0, len(inner), 64):
block = inner[block_off:block_off + 64]
cipher_words = [int.from_bytes(block[i:i + 4], "big") for i in
range(0, 64, 4)]

plain_words = decrypt_block(cipher_words, h_words)
for word in plain_words:
out.extend(word.to_bytes(4, "big"))
h_words = [cipher_words[i] ^ cipher_words[i + 8] for i in range(8)]

pad = out[-1]
if not (1 <= pad <= 64 and out.endswith(bytes([pad]) * pad)):
raise ValueError("invalid padding after VM decrypt")

out = out[:-pad]
wav_out_path.write_bytes(out)
return bytes(out)

def validate_wav(wav_path: Path) -> tuple[int, int, int, int]:
with wave.open(str(wav_path), "rb") as wav_file:
return (
wav_file.getnchannels(),
wav_file.getsampwidth(),
wav_file.getframerate(),
wav_file.getnframes(),
)

def verify_with_native(wav_path: Path, enc_path: Path, native_path: Path =
NATIVE_MODULE) -> bool:
js = """
const fs = require('fs');
const mod = require(process.argv[1]);
const wav = fs.readFileSync(process.argv[2]);
const expected = fs.readFileSync(process.argv[3]);
const got = mod.vmEncrypt(wav);
process.stdout.write(Buffer.compare(got, expected) === 0 ? 'OK' : 'FAIL');
"""
result = subprocess.run(
["node", "-e", js, str(native_path.resolve()),
str(wav_path.resolve()), str(enc_path.resolve())],
capture_output=True,
text=True,
check=False,
)
if result.returncode != 0:
raise RuntimeError(result.stderr.strip() or "native verification
failed")
return result.stdout.strip() == "OK"

def compress_literal_only(data: bytes) -> bytes:
out = bytearray()
pos = 0

while pos < len(data):
chunk = data[pos:pos + 32]
out.append(len(chunk) - 1)
for i, b in enumerate(chunk):
out.append(b ^ ((i * 0x11) & 0xFF))
pos += len(chunk)
return bytes(out)

def compress_greedy(data: bytes) -> bytes:
recent: dict[bytes, collections.deque[int]] =
collections.defaultdict(collections.deque)
out = bytearray()
pos = 0
size = len(data)

while pos < size:
best_kind = "lit"
best_len = 1
best_flag = 0
best_off = 0

run = 1
while pos + run < size and run < 34 and data[pos + run] == data[pos]:
run += 1
if run >= 3:
best_kind = "rep"
best_len = run

for step, flag in ((1, 0), (2, 1)):
run = 1
while pos + run < size and run < 34 and ((data[pos + run] -
data[pos + run - 1]) & 0xFF) == step:
run += 1
if run >= 3 and run > best_len:
best_kind = "arith"
best_len = run
best_flag = flag

if pos + 4 <= size:
key = bytes(data[pos:pos + 4])
best_back_len = 0
best_back_off = 0
for prev in reversed(recent.get(key, ())):
if pos - prev > 1024:
break
run = 4

while pos + run < size and prev + run < pos and run < 19 and
data[prev + run] == data[pos + run]:
run += 1
if run > best_back_len:
best_back_len = run
best_back_off = pos - prev
if best_back_len >= 4 and best_back_len > best_len:
best_kind = "back"
best_len = best_back_len
best_off = best_back_off

if best_kind == "rep":
out.append((1 << 6) | (best_len - 3))
out.append((((data[pos] << 1) | (data[pos] >> 7)) & 0xFF) ^ 0x5C)
elif best_kind == "arith":
out.append((2 << 6) | (best_flag << 5) | (best_len - 3))
out.append(data[pos])
elif best_kind == "back":
off = best_off - 1
out.append((3 << 6) | (((off >> 8) & 0x3) << 4) | (best_len - 4))
out.append(off & 0xFF)
else:
end = pos + 1
while end < size and end - pos < 32:
stop = False

run = 1
while end + run < size and run < 3 and data[end + run] ==
data[end]:
run += 1
if run >= 3:
stop = True

if not stop:
for step in (1, 2):
run = 1
while end + run < size and run < 3 and ((data[end +
run] - data[end + run - 1]) & 0xFF) == step:
run += 1
if run >= 3:
stop = True
break

if not stop and end + 4 <= size:
key = bytes(data[end:end + 4])
for prev in reversed(recent.get(key, ())):
if end - prev > 1024:

break
if data[prev:prev + 4] == data[end:end + 4]:
stop = True
break

if stop:
break
end += 1

chunk = data[pos:end]
out.append(len(chunk) - 1)
for i, b in enumerate(chunk):
out.append(b ^ ((i * 0x11) & 0xFF))
best_len = len(chunk)

for p in range(pos, min(pos + best_len, size)):
if p + 4 <= size:
key = bytes(data[p:p + 4])
dq = recent[key]
dq.append(p)
while dq and p - dq[0] > 1024:
dq.popleft()

pos += best_len

return bytes(out)

def build_sumv(payload: bytes, compressed: bytes, version: int = 1,
format_code: int = 1) -> bytes:
out = bytearray(b"SUMV")
out.extend(bytes([version, 0, format_code, 0]))
out.extend(struct.pack("<I", len(payload)))
out.extend(struct.pack("<I", len(compressed)))
out.extend(compressed)
return bytes(out)

def main() -> None:
enc_path = Path("ddd.su_mv_enc")
wav_path = Path("recovered_payload.wav")

payload = decrypt_vm_encryptor_output(enc_path, wav_path)
print(f"[+] recovered payload: {wav_path} ({len(payload)} bytes)")
wav_md5 = hashlib.md5(payload).hexdigest()
print(f"[+] payload md5: {wav_md5}")

channels, sampwidth, framerate, nframes = validate_wav(wav_path)
print(

"[+] wav info:",
f"channels={channels}",
f"sampwidth={sampwidth}",
f"framerate={framerate}",
f"frames={nframes}",
)

if not verify_with_native(wav_path, enc_path):
print("[-] native round-trip verification failed", file=sys.stderr)
raise SystemExit(1)
print("[+] native round-trip verification passed")
print(f"[+] submit md5: {wav_md5}")
print(f"[+] flag-style candidate: SUCTF{{{wav_md5}}}")

enc_payload = rc4_crypt(payload)

literal_comp = compress_literal_only(enc_payload)
literal_sumv = build_sumv(payload, literal_comp)
literal_path = Path("recovered_candidate_literal.su_mv")
literal_path.write_bytes(literal_sumv)
print(f"[+] literal candidate md5:
{hashlib.md5(literal_sumv).hexdigest()}")

greedy_comp = compress_greedy(enc_payload)
greedy_sumv = build_sumv(payload, greedy_comp)
greedy_path = Path("recovered_candidate_greedy.su_mv")
greedy_path.write_bytes(greedy_sumv)
print(f"[+] greedy candidate md5: {hashlib.md5(greedy_sumv).hexdigest()}")

if __name__ == "__main__":
main()

SUCTF{16ac79d3510d6ea4b5338fade80459b8}
```

## 方法总结
- 核心技巧：Electron + native addon VM 逆向
- 识别信号：asar 中有前端逻辑，真正加密/解密在 `.node` 原生模块或 VM 中。
- 复用要点：先解包 asar 定位调用链，再逆 native 模块的 VM/opcode，恢复数据流并输出原始媒体。
