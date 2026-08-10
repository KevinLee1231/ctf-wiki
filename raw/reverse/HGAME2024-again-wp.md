# again！

## 题目简述

附件是 PyInstaller 打包程序，内部包含一个无法被 `pycdc` 完整反编译的 `bin1.pyc`，以及被加密后无法直接执行的第二阶段 PE。第一阶段字节码从自身头部导出一个 MD5 字符串；该 MD5 的十六进制文本作为循环 XOR 密钥，恢复第二阶段 PE。第二阶段再使用修改过 Delta 常量的 Block TEA/XXTEA 解密 8 个 32 位整数，得到 flag。

## 解题过程

### 解包并反汇编 Python 字节码

先用 `pyinstxtractor` 解开 PyInstaller 包。`pycdc` 无法完整还原 `bin1.pyc` 时，不应依赖残缺伪代码；使用与该 `.pyc` 版本匹配的 Python，通过 `marshal` 载入 code object 并交给 `dis`：

```python
import dis
import marshal

with open("bin1.pyc", "rb") as file:
    file.seek(16)
    code = marshal.loads(file.read())

dis.dis(code)
```

这里的 `16` 是该题 `.pyc` 的头部长度；若本地版本不同，应先识别 magic number 和头结构，不能机械套用偏移。根据字节码还原出的第一阶段逻辑为：

```python
import hashlib

print('you should use this execute file to decrypt "bin2"')
print("hint:md5")

result = bytearray()
data = bytearray(open("bin1.pyc", "rb").read())
text = "jkasnwojasd"

for i in range(15):
    data[i] = (
        (data[i] + data[i % 6])
        ^ (ord(text[i % 6]) + ord(text[i % len(text)]))
    ) % 256
    result.append(data[i])

md5_key = hashlib.md5(bytes(result)).hexdigest()
print(md5_key)
```

注意循环会原地修改 `data`，所以当 $i\geq6$ 时，`data[i % 6]` 已经是前面更新后的值。运行得到：

```text
a405b5d321e446459d8f9169d027bd92
```

### 循环 XOR 恢复第二阶段 PE

加密文件在十六进制视图中反复出现与 MD5 文本相同的字节。用密钥首字节和文件首字节 XOR 得到 `M`，第二字节同理得到 `Z`，从而验证它是以 MD5 十六进制文本为循环密钥的 XOR，而不是把摘要解析为 16 个原始字节。

```python
from pathlib import Path

encrypted = Path("again!.exe").read_bytes()
key = b"a405b5d321e446459d8f9169d027bd92"

decrypted = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(encrypted)
)

assert decrypted.startswith(b"MZ")
Path("bin2.exe").write_bytes(decrypted)
```

官方脚本把加密载荷命名为 `again!.exe`；若解包后的实际文件名是 `bin2`，只需替换输入文件名，算法不变。

### 解密 Block TEA 数据

恢复出的 `bin2.exe` 内含标准 Block TEA 结构，但 Delta 被改成 `0x7937B99E`。密文和 128 位密钥分别为：

```text
v = 506fb5c3 b9358f45 c91ae8c7 3820e280
    d13aba83 975cf554 4352036b 1cd20447

key = 00001234 00002341 00003412 00004123
```

按反编译代码的无符号 32 位语义实现解密，并以小端序拼接每个 word：

```python
MASK = 0xFFFFFFFF
DELTA = 0x7937B99E

words = [
    0x506FB5C3,
    0xB9358F45,
    0xC91AE8C7,
    0x3820E280,
    0xD13ABA83,
    0x975CF554,
    0x4352036B,
    0x1CD20447,
]
key = [0x1234, 0x2341, 0x3412, 0x4123]

n = len(words)
rounds = 6 + 52 // n
total = (rounds * DELTA) & MASK
y = words[0]

for _ in range(rounds):
    e = (total >> 2) & 3

    for p in range(n - 1, 0, -1):
        z = words[p - 1]
        mx = (
            (((z >> 5) ^ ((y << 2) & MASK))
             + ((y >> 3) ^ ((z << 4) & MASK)))
            ^ ((total ^ y) + (key[(p & 3) ^ e] ^ z))
        ) & MASK
        words[p] = (words[p] - mx) & MASK
        y = words[p]

    z = words[n - 1]
    p = 0
    mx = (
        (((z >> 5) ^ ((y << 2) & MASK))
         + ((y >> 3) ^ ((z << 4) & MASK)))
        ^ ((total ^ y) + (key[(p & 3) ^ e] ^ z))
    ) & MASK
    words[0] = (words[0] - mx) & MASK
    y = words[0]
    total = (total - DELTA) & MASK

plaintext = b"".join(word.to_bytes(4, "little") for word in words)
print(plaintext.decode())
```

输出为：

```text
hgame{btea_is_a_hard_encryption}
```

这里必须在每轮运算后按 $2^{32}$ 截断；Python 整数不会像 C 的 `uint32_t` 一样自动回绕。最终按小端序输出，则来自原程序逐次打印 `v[i] & 0xff` 并右移 8 位的行为。

## 方法总结

- PyInstaller 解包后，反编译器失败不代表 `.pyc` 不可分析；`marshal` 加载配合 `dis` 能保留真实字节码语义。
- 自修改数组循环要注意原地更新造成的数据依赖，尤其是后续索引重新引用前六个已修改字节的情况。
- 猜测 XOR 时应先用文件魔数验证：循环密钥首两字节恢复 `MZ` 是强证据，再对完整文件解密。
- MD5 既可能作为 16 字节摘要，也可能以 32 字符十六进制文本参与运算；本题使用后者。
- 识别 TEA 家族后仍要核对 Delta、轮数、端序和 32 位回绕，不能直接套标准 XXTEA 参数。
