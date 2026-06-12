# baby_spear

## 题目简述

本题模拟 Office 宏钓鱼到勒索加密的攻击链：残缺的 Office 文档中包含被隐藏和混淆的 VBA 宏，宏负责校验 `lsc.key` 并释放后续 PE，PE 再用 AES-ECB 加密/解密目标文件。核心目标是恢复宏逻辑、取出 PE，并利用文件时间生成 AES key 解密 `flag.png.encrypted`。

关键机制包括 VBA purging、EvilClippy 隐藏模块、macro_pack/junk code 混淆、宏中的大整数校验，以及 PE 中由 `time()`/`srand()` 派生的 32 字节 AES key。

## 解题过程

本题模拟 Office 文档钓鱼攻击：宏在打开文档后释放并执行恶意程序，后续 PE 对目标文件做 AES 加密。难点不在 PE，而在先把隐藏、purge 和混淆过的 VBA 宏恢复出来。

宏代码无法直接在 Office 的 VBA 编辑器中观察和调试，主要隐藏手法包括：

1. EvilClippy 的 `-g` 参数隐藏 GUI 中的模块信息。
2. 打开文档执行后 VBA 代码自删除。
3. VBA purging 删除 VBA 流中的 P-code 数据，`_VBA_PROJECT` 附近的 7 字节特征可以作为识别信号。

由于 P-code 已被清除，不能只依赖 `olevba` 这类默认解析路径。应改用能解压/还原 VBA 压缩流的工具，例如 `oledump.py`、OfficemMalScanner，或者 Kavod 的 VBA compression 库 <https://github.com/rossknudsen/Kavod.Vba.Compression>。使用 `oledump.py -v` 可以列出所有模块，带 `M` 标志的是宏模块；本题中模块 15、16、18、19、21 是重点，继续用 `-s <index>` 提取源码。

宏打开即执行，入口是 `AutoOpen`，可在模块 19 找到。该模块充满 junk code，并且配合 `On Error Resume Next` 让无意义调用出错后继续执行。跳过 junk 后可看到关键调用：

```vb
buakhctj = bSa7akz.cestitgm(elasbhnp, idhkqsrq)
```

该调用进入模块 15。模块 15 基本没有 junk，适合复制到新 Office 文档宏里做黑盒调用。由于输入输出完全可控，可以确认它实际实现的是大整数运算。结合对 `word.exe` 设置 IFEO 并在文件/API 访问处下断，可发现宏会读取 `lsc.key`，最终还原出校验逻辑：

```python
q = lsc_key
p, M, n = hardcoded_values

if (p * q) % M == n:
    print("check passed")
```

反解就是求模逆：

```python
inv = inverse(p, M)
q = (n * inv) % M
```

得到 `lsc.key = ID0ntWantT0SetTheWor1dOnFIRE`。不过即使 `lsc.key` 通过，PE 也不会正常 drop，题干也提示文档不完整，可以理解为取证时文件破损。继续用 API 断点，在 `CryptDecrypt` 附近下断即可取得被释放的 PE。

PE 部分是 AES-ECB。hint 表示不需要暴力爆破，压缩包里的加密文件生成时间和程序中的 `time()` / `srand()` 派生 key 能对应上。用 Windows 的 `ucrtbase.dll` 复现 `rand()`，按文件时间生成 32 字节 key 后解密 `flag.png.encrypted`：

```python
from Crypto.Cipher import AES
from ctypes import cdll
import time

def gen_key():
    timestr = "2021-03-04 20:53:16"  # UTC+8
    timestruct = time.strptime(timestr, "%Y-%m-%d %H:%M:%S")
    timestamp = int(time.mktime(timestruct))

    lib = cdll.LoadLibrary("C:\\Windows\\System32\\ucrtbase.dll")
    lib.srand(timestamp)
    table = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    return "".join(table[lib.rand() % 62] for _ in range(32))

with open("flag.png.encrypted", "rb") as f:
    buf = f.read()

key = gen_key().encode()
cipher = AES.new(key, mode=AES.MODE_ECB)

tail = buf[len(buf) - len(buf) % 16:]
body = buf[:len(buf) - len(buf) % 16]

with open("flag.png", "wb") as f:
    f.write(cipher.decrypt(body))
    f.write(tail)
```

## 方法总结

- 核心技巧：Office 宏取证与恶意链还原，先绕过 VBA purging 和宏混淆提取源码，再通过 API 断点恢复释放的 PE，最后分析 PE 的 AES key 生成逻辑。
- 识别信号：`_VBA_PROJECT`、宏模块无法从 VBA 编辑器直接查看、`AutoOpen` 自动执行、大量 `On Error Resume Next` junk code，以及对 `CryptDecrypt` 等 API 的调用。
- 复用要点：遇到 P-code 被清除的文档不要只依赖 `olevba` 默认提取；可以用 `oledump.py -v/-s`、OfficemMalScanner 或 Kavod 这类解压/解析工具，并结合 IFEO/API 断点观察宏实际访问的文件和释放行为。

