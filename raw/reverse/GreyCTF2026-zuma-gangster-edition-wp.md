# Zuma Gangster Edition

## 题目简述

题目提供一个运行于 Wine 的 Zuma 游戏。表面目标是向同色球串发射球并触发消除，真正的校验逻辑却藏在原生可执行文件中：每次消除会依据“颜色 + 消除后长度”产生一至三个字节，并写入一个 24 字节的 `forge` 结构。只有把该结构拼成满足内部字段约束的正确值，程序才会用它派生 AES 密钥并解出 flag。

因此本题的决定性障碍是恢复游戏二进制中的状态转换、字节映射和写入位置，归类为 Reverse。背景、青蛙和球的 PNG 只是界面素材，解题所需的颜色与球串信息可以无损写成文本，故不在 WP 中保留装饰性截图。

## 解题过程

从二进制字符串和符号入手，可以定位 `game_init_chain`、`game_fire_shot`、`game_check_and_clear`、`game_remove_balls` 以及胜利路径中的 `[win] forged_key`。程序使用六种颜色，官方脚本按如下顺序编号：

```python
COLORS = "RBGYPO"  # Red, Blue, Green, Yellow, Purple, Orange
```

当原球串中已有 $n$ 个连续同色球时，再射入一球会形成长度 $n+1$ 的消除组。只有长度 3 至 8 有对应编码；例如红色、蓝色和绿色的部分映射为：

```text
R x3 -> 00       R x6 -> 60b0       R x7 -> 04a0c0
B x3 -> 20       B x7 -> 24a242
G x3 -> 40       G x5 -> 42a4       G x6 -> 70b4
Y x3 -> 08       Y x5 -> 0a04
P x3 -> 28
O x3 -> 48       O x4 -> 58
```

前八次单字节消除构成 `forge[0:8]`。后续多字节结果并非依次追加，而是由颜色和长度决定写入起点；后写入的数据可以覆盖先前字节。反编译后可将这一规则还原为：

```python
# BODY_SLOT[color][clear_len - 3]
BODY_SLOT = {
    "R": [16, 19, 10, 17, 13, 8],
    "B": [16, 22, 10, 17, 8, 13],
    "G": [19, 22, 10, 20, 8, 13],
    "Y": [22, 23, 12, 17, 8, 13],
    "P": [19, 23, 12, 20, 8, 13],
    "O": [16, 23, 12, 20, 8, 13],
}
```

初始球串共有 111 个球：

```text
RROPPPPGBYYBRRRYROOPGGOBBBBGYPPGYYYPRBBYRRYOOOPGRROBBBBBBGBBORGGGGOYYYYPRRRRRROGGGGGBGGYROOBOOBPRRRRRBOOORPPBYY
```

扫描连续同色段，就能筛掉球串中根本无法触发的映射。对每个仍可用的写入位置枚举一种消除方式，按实际覆盖顺序构造 `forge`，再检查二进制中的两个快速约束：

```python
cnt = int.from_bytes(forge[0x10:0x14], "little", signed=True)
if cnt < 8:
    reject()
if forge[0x14] & 0x40 == 0:
    reject()
```

满足约束的候选还不能直接视为答案。程序以完整 24 字节 `forge` 为 PBKDF2 输入，盐为 `no_LLM_to_help_u`，执行 2,000,000 次 HMAC-SHA256，取 24 字节作为 AES-192 密钥；随后使用全零 IV 的 CBC 模式解密内置密文，并验证 PKCS#7 填充。可用下面的核心逻辑逐个确认候选：

```python
import hashlib
from cryptography.hazmat.primitives import padding
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

key = hashlib.pbkdf2_hmac(
    "sha256", bytes(forge), b"no_LLM_to_help_u", 2_000_000, dklen=24
)
decryptor = Cipher(algorithms.AES(key), modes.CBC(b"\x00" * 16)).decryptor()
padded = decryptor.update(ciphertext) + decryptor.finalize()
unpadder = padding.PKCS7(128).unpadder()
plaintext = unpadder.update(padded) + unpadder.finalize()
```

正确路径依次向下列 18 个球组射入同色球：

```text
目标组：R3 Y3 O3 G3 P3 B3 R3 R3 B7 G5 Y5 R7 G6 O3 R6 O4 P3 Y3
弹药：  R  Y  O  G  P  B  R  R  B  G  Y  R  G  O  R  O  P  Y
发射：  00 08 48 40 28 20 00 00 24a242 42a4 0a04 04a0c0 70b4 48 60b0 58 28 08
```

这 18 次消除产生的顺序字节流为：

```text
000848402820000024a24242a40a0404a0c070b44860b0582808
```

考虑写入槽位和覆盖关系后，真正参与密钥派生的结构是：

```text
000848402820000024a242a40a04a0c04860b02870b40858
```

其中 `forge[0x10:0x14]` 按小端解释为 `0x28b06048`，且 `forge[0x14] = 0x70`，两个快速约束均成立。按上述弹药顺序完成实际游戏，胜利路径即可解密得到：

```text
grey{1_S7AY_s7Ra1ghT_1_STAy_Bad_1_kEeP_My_F1n9er_0N_7He_TR199Er}
```

## 方法总结

本题把逆向出的数据流约束藏在了正常游戏操作后面。有效做法是先从消除函数恢复“颜色、长度到字节”的映射，再恢复各编码对 24 字节结构的写入位置，最后只枚举初始球串实际支持的连续段，并用内部字段和真实解密结果双重验真。

需要特别区分“消除时按顺序产生的 26 字节流”和“覆盖写入后留下的 24 字节 `forge`”：PBKDF2 使用的是后者。只抄射击顺序而不分析槽位覆盖，无法解释密钥为何正确；只满足两个字段约束而不执行 AES 解密，也会保留大量伪候选。
