# Q(≧▽≦)T

## 题目简述

附件中的 `CrackMe.exe` 是一个 Qt 图形程序，要求输入用户名和密码。校验分为两层：用户名长度必须为 4，随后计算其 SHA-256 与内置摘要比较；用户名通过后，再用该用户名作为 RC4 密钥处理密码，并与内置密文比较。

压缩包还包含 `main.o`、`mainwindow.o` 等构建残留，但它们并非当前可执行文件对应的对象文件。例如 `mainwindow.o` 只有单密码框，内置摘要也是另一组 `68934a...`。实际程序包含用户名、密码两个输入框，并在 `.rdata` 中保存 `c942...` 和 `af33...` 两个常量，因此分析时应以 `CrackMe.exe` 的执行逻辑为准。

## 解题过程

### 1. 定位用户名校验

在按钮回调中可以看到程序先读取两个 `QLineEdit`，并检查用户名长度是否为 4。随后调用 `QCryptographicHash::hash`；算法枚举值为 `4`，对应 SHA-256。

程序中的目标摘要为：

```text
c94201919ec7463313c747d0a27fabcabf1400fa1e9a64d36a6b1a7e7b12ae68
```

原 PDF 用 `kkkk` 的摘要 `503afc4b...` 识别算法，但相邻栈数据对应的是大写 `KKKK`；两者摘要不同，不能混用。这里直接对 1～4 位英文字母穷举，并用实际目标值验证：

```python
import hashlib
import itertools
import string


target = "c94201919ec7463313c747d0a27fabcabf1400fa1e9a64d36a6b1a7e7b12ae68"

for length in range(1, 5):
    for chars in itertools.product(string.ascii_letters, repeat=length):
        candidate = "".join(chars)
        digest = hashlib.sha256(candidate.encode()).hexdigest()
        if digest == target:
            print(candidate)
            raise SystemExit
```

输出：

```text
Kath
```

也可以直接复核：

```python
import hashlib

assert hashlib.sha256(b"Kath").hexdigest() == (
    "c94201919ec7463313c747d0a27fabcabf1400fa1e9a64d36a6b1a7e7b12ae68"
)
```

### 2. 还原密码校验

用户名通过后，程序用一个 256 字节状态数组执行密钥调度：

```text
for i = 0..255:
    j = (j + S[i] + key[i mod key_len]) mod 256
    swap(S[i], S[j])
```

随后逐字节更新两个索引、交换状态并异或密钥流。这正是标准 RC4 的 KSA 与 PRGA，IDA 中的循环也能看到两个索引取模 256、交换数组元素和最终异或。

可执行文件中保存的十六进制密文为：

```text
af33da5e152c15863b3a03c87601899a37d51b8b8168f65aca65352d3669e91959300ccb
```

RC4 加解密是同一过程，以用户名 `Kath` 为密钥解密即可。下面的脚本完整实现 KSA 和 PRGA，不依赖第三方库：

```python
def rc4(data: bytes, key: bytes) -> bytes:
    state = list(range(256))
    j = 0

    for i in range(256):
        j = (j + state[i] + key[i % len(key)]) & 0xFF
        state[i], state[j] = state[j], state[i]

    i = 0
    j = 0
    output = bytearray()
    for value in data:
        i = (i + 1) & 0xFF
        j = (j + state[i]) & 0xFF
        state[i], state[j] = state[j], state[i]
        key_byte = state[(state[i] + state[j]) & 0xFF]
        output.append(value ^ key_byte)

    return bytes(output)


ciphertext = bytes.fromhex(
    "af33da5e152c15863b3a03c87601899a37d51b8b8168f65aca65352d3669e91959300ccb"
)
password = rc4(ciphertext, b"Kath").decode()
print(password)
print(f"0xGame{{{password}}}")
```

输出：

```text
ce5e5621-3d6b-4429-b72a-957abf353390
0xGame{ce5e5621-3d6b-4429-b72a-957abf353390}
```

程序输入的用户名为 `Kath`，密码为解出的 UUID；平台提交时按比赛格式补上 `0xGame{}`。

## 方法总结

本题的完整数据流是“4 字符用户名 → SHA-256 比较 → 用户名作为 RC4 密钥 → 解出 36 字符密码”。识别 Qt 程序时，`QLineEdit::text`、`QCryptographicHash::hash`、`QByteArray::fromHex` 等符号可以快速标出输入、哈希和十六进制常量。

附件中存在与最终程序不一致的构建残留，因此不能仅凭文件名认定对象文件就是源码。应交叉核对界面控件、内置常量和实际导入函数；本题中 `CrackMe.exe` 的 `c942...`、`af33...` 与 PDF 的校验链能够互相印证，而通用示例对象文件应排除。
