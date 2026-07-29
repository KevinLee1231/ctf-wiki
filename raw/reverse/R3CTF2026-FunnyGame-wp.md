# FunnyGame

## 题目简述

题目提供一个约 131 MB 的 Windows x64 游戏附件 `FunnyGame.7z`，仓库说明它是静态离线逆向题，flag 完整地藏在游戏二进制及资源中。附件的 SHA-256 为：

```text
b3c33ef4217ec5c83a3f225e13b78f9c190114563a91e9f35cfa4aac69bbd43e
```

解压后的目录很像标准 Unity IL2CPP 游戏：

```text
FunnyGame.exe
GameAssembly.dll
UnityPlayer.dll
FunnyGame_Data/
├── il2cpp_data/Metadata/global-metadata.dat
├── resources.assets
├── resources.assets.resS
└── Plugins/x86_64/audio_core.dll
```

但 Unity 只是第一层伪装。`GameAssembly.dll` 中的 checker、ciphertext 和大量命名可信的类最终只会生成 `r0ctf`、`r1ctf`、`r2ctf`、`r4ctf` 或 `r5ctf` 前缀的假 flag。真正的执行链是：

1. 解包 `FunnyGame.exe`，得到经过定制的 Godot 4.6.2 引擎；
2. 从 `FunnyGame_Data/resources.assets.resS` 的偏移 `0x100` 提取加密 Godot PCK；
3. 解密 PCK，并还原经过修改的 GDScript 字节码；
4. 从 `FlagSeal.gdc` 取得前两段 flag；
5. 逆向作为 Godot GDExtension 加载的 `audio_core.dll`，取得最后一段。

决定性障碍是识别伪装层并还原自定义加载器、资源格式和字节码，因此归入 `reverse`，而不是按游戏表面归入杂项。

## 解题过程

### 识别 Unity 诱饵

先检查主程序和原生插件。`FunnyGame.exe` 与 `audio_core.dll` 都只有三个 PE section，入口附近是解包 stub；后者主要导入 `LoadLibraryA`、`GetProcAddress`、`VirtualProtect` 等加载和内存管理函数，不像正常的音频库。

Unity 元数据中虽然存在许多貌似与节奏判定有关的类，例如：

```text
ApproachRingAudit
ChartFalseVault
ChartLaneMath
ChartLongHashGate
ChartRuntimeBootstrap
TimingWindowNode00 ... TimingWindowNode63
```

但这些路径没有任何一条原生生成正确的 `r3ctf{...}` 前缀，其中一个相当逼真的输出甚至是：

```text
r0ctf{un1ty_d0es_n0t_h0ld_th3_r1ng}
```

不能把它的前缀手工改成 `r3ctf`：这一层的校验逻辑和 ciphertext 本身都是诱饵。更可靠的转向证据是，解包后的 `FunnyGame.exe` 含有以下引擎标识：

```text
Godot Engine v4.6.2.stable.custom_build
D:\Workspace\chall\r3ctf2026\Reverse\MyGodot\godot\modules\gdscript\gdscript.h
```

解包后的 `audio_core.dll` 则出现 `native_seal`、`godot::A9` 等名称，并导出 `audio_core_library_init`。这说明 Unity 文件只是容器，实际游戏运行时是 Godot。

### 提取并解密 Godot PCK

读取 `resources.assets.resS` 可在偏移 `0x100` 看到 Godot PCK 魔数：

```text
00000100  47 44 50 43 03 00 00 00 ...
          G  D  P  C
```

本地附件中这一偏移已直接核对。可以从这里切出 PCK：

```python
from pathlib import Path

src = Path("FunnyGame_Data/resources.assets.resS").read_bytes()
assert src[0x100:0x104] == b"GDPC"
Path("funnygame.pck").write_bytes(src[0x100:])
```

PCK 头部给出的格式版本为 3，Godot 版本为 4.6.2，flags 为 3，表示目录和文件条目都经过加密。定制引擎中的 32 字节密钥为：

```text
3a00dbacb3316f082fdbb88c7fd4a6b1c1333ef17203b5c70619813ca6efa650
```

每个加密条目的布局为：

```text
+0x00  明文 MD5             16 字节
+0x10  明文长度             uint64 little-endian
+0x18  IV                   16 字节
+0x28  ciphertext           向 16 字节对齐
```

实现使用的是 AES-256-CFB128，而不是包装函数名称容易让人误判的 CBC。单个条目可按下面的逻辑恢复：

```python
import hashlib
import struct
from Crypto.Cipher import AES

key = bytes.fromhex(
    "3a00dbacb3316f082fdbb88c7fd4a6b1"
    "c1333ef17203b5c70619813ca6efa650"
)

md5_expected = blob[:16]
length = struct.unpack_from("<Q", blob, 16)[0]
iv = blob[24:40]
aligned_length = (length + 15) & ~15
ciphertext = blob[40:40 + aligned_length]

plaintext = AES.new(
    key,
    AES.MODE_CFB,
    iv=iv,
    segment_size=128,
).decrypt(ciphertext)[:length]

assert hashlib.md5(plaintext).digest() == md5_expected
```

MD5 是判断模式是否正确的直接证据：使用 CFB128 时，解出的明文摘要与条目头完全一致。解密后的 42 个文件中，关键对象包括：

```text
Main.tscn
Interface.tscn
Hitball.tscn
Scripts/Main.gdc
Scripts/FlagSeal.gdc
Scripts/Hitball.gdc
bin/native_seal.gdextension
project.binary
```

`native_seal.gdextension` 也确认了原生插件的加载关系：

```ini
[configuration]
entry_symbol = "audio_core_library_init"
compatibility_minimum = "4.1"

[libraries]
windows.release.x86_64 = "res://FunnyGame_Data/Plugins/x86_64/audio_core.dll"
```

### 还原自定义 GDScript 字节码

普通 GDScript 反编译器无法直接处理 `FlagSeal.gdc`，因为题目修改了字节码格式：

```text
magic   = ABAB
version = 0x43544612
```

定制主要发生在三个位置：

1. identifier 以 UTF-32 codepoint 保存，并按 identifier 序号和字符序号生成密钥流；
2. Variant constant 前增加 `0xc7` marker，并按 constant 序号加密；
3. token ID 在写入前经过置换。

token 的逆映射可写成：

```python
def decode_token(raw):
    value = raw & 0x7f
    if value == 0:
        return 0
    if value >= 100:
        return 98
    return ((2 * ((value - 18) % 99)) % 99) + 1
```

identifier 与 constant 的混淆都使用同一组 32 位混合常数 `0x7FEB352D` 和 `0x846CA68B`，种子再分别混入 identifier/字符序号或 constant 序号。解开后可看到这些关键标识：

```text
EXTREME_LEVEL
NORMAL_LEVEL
EXTREME_HIT_COUNT
EXTREME_TRACE
PART1_CIPHER
PART2_CIPHER
initial_trace
feed_hit
try_open_part1
try_open_part2
_decrypt
_next32
_mix32
```

恢复出的关键常量为：

```text
NORMAL_LEVEL      = 6
EXTREME_LEVEL     = 11
EXTREME_HIT_COUNT = 64
EXTREME_TRACE     = 2271314799
GRADE_CODE        = {"Good": 1, "Great": 2, "Perfect": 3}
```

`try_open_part1` 检查普通难度的分数、miss 和 combo 状态；`try_open_part2` 则要求 level 11、64 次命中、无 miss、combo 64 且 trace 等于 `2271314799`。无需实际完成整张谱面：静态恢复两个 ciphertext 和 `_decrypt` 例程后，可以直接解出：

```text
r3ctf{0dd_74p5_7h3n_5113n
c3_f0110w_7h3_0r817_70_un
```

### 恢复原生插件中的尾段

解包 `audio_core.dll` 后，Godot 类 `A9` 暴露五个方法：

```text
f0()                         重置状态
f1(int, int, ..., int)       校验七个整数
f2(int, int)                 校验后续状态
f3(Vector2, float)           接收音符位置和时间戳
f4() -> String               解开最后一个片段
```

`f3` 会把点击位置换算为环形区域中的 sector，要求八个音符依次落在：

```text
0, 4, 7, 3, 6, 2, 5, 1
```

其余约束是半径约为 265 至 390、相邻音符间隔约为 0.09 至 0.72 秒，并恰好接受八次有效输入。通过 `f1`、`f2` 和这组轨道状态后，`f4` 解出：

```text
10ck_7h3_f1n41_n073}
```

这不是独立 flag，而是前两段中 `un` 的后续，使整句话成为：

```text
odd taps then silence follow the orbit to unlock the final note
```

依次拼接三段，得到静态 flag：

```text
r3ctf{0dd_74p5_7h3n_5113nc3_f0110w_7h3_0r817_70_un10ck_7h3_f1n41_n073}
```

公开复现还提供了去混淆算法和制品校验脚本；本文已经展开其 PCK 布局、密钥、字节码改动、状态约束和全部 flag 片段，原始记录仅作为证据来源保留：[FunnyGame — R3CTF 2026](https://github.com/RahmatHadinata23758051/Writeup/blob/main/R3CTF/misc/FunnyGame/Readme.md)。

## 方法总结

- 文件布局只能作为初始线索，不能代替执行链判断。本题的 Unity 元数据和假 checker 足够完整，但正确前缀、入口 stub、引擎字符串与 GDExtension 符号共同证明实际运行时是 Godot。
- 处理自定义资源格式时，应同时恢复结构和校验条件。`GDPC` 偏移定位 PCK，条目 MD5 则把 AES-CFB128 从错误的候选模式中确定下来。
- 修改版字节码无需先做完整反编译器。优先恢复 identifier、constant 与 token 三类信息，已经足以重建 flag 解密所需的数据流和状态条件。
- 分段 flag 必须结合调用关系确定顺序。`FlagSeal.gdc` 给出开头与中段，`audio_core.dll` 的 `f4` 只给尾段，任何单独片段都不能当作最终答案。
