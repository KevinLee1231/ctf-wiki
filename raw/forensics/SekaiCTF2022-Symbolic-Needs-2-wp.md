# Symbolic Needs 2

## 题目简述

第二问要求恢复以太坊地址：

```text
0xACa5872e497F0Cc626d1E9bA28bAEC149315266e
```

对应的私钥。内存中的进程参数保存了一段 Base32 数据，曾由 `ncat` 发送并解码成 Python 字节码。反汇编 `.pyc` 可以取出硬编码整数及其助记词生成逻辑，随后按标准 BIP39/BIP44 派生目标钱包。

## 解题过程

使用第一问准备好的 Linux 符号表，枚举完整进程命令行：

```console
python3 vol.py -f dump.mem linux.psaux
```

PID 1878 的参数包含：

```text
ncat -lvnp 1234 -c echo <BASE32> |
base32 -d > file.pyc
```

进程参数本身就是证据：复制其中的 Base32 串并解码，即可重建 `file.pyc`。

`.pyc` 的前 16 字节是 Python 版本相关头部，后面是 marshal 序列化的代码对象。用相同主版本的 Python 反汇编：

```python
import dis
import marshal

with open("file.pyc", "rb") as stream:
    stream.seek(16)
    code = marshal.load(stream)

dis.dis(code)
```

反汇编结果表明程序：

1. 读取命令行中的 `password`，但之后根本不使用；
2. 加载 `bip39list.txt`；
3. 把一个硬编码大整数转换成二进制；
4. 左侧补零到 12 的倍数；
5. 每 12 位取一个整数，减 1 后作为单词表下标；
6. 无论输入什么，最后都只打印 `Wrong`。

硬编码整数为：

```text
75673125099835840306362297188218306412669859836254678874904603942583570317024638985472
```

按字节码中的真实逻辑恢复单词：

```python
code = 75673125099835840306362297188218306412669859836254678874904603942583570317024638985472

bits = bin(code)[2:]
bits = bits.zfill(
    len(bits) + (12 - len(bits) % 12)
)

mnemonic = [
    words[int(bits[i:i + 12], 2) - 1]
    for i in range(0, len(bits), 12)
]
```

得到 24 个单词：

```text
evidence leopard solution layer legend danger
orient project silver flower wrong path
stove throw fortune report nuclear old
target exact broom hawk toss paper
```

接下来无需依赖网页钱包。先按 BIP39 计算种子：

$$
\text{seed}=\operatorname{PBKDF2\text{-}HMAC\text{-}SHA512}(\text{mnemonic},\text{"mnemonic"},2048).
$$

再按 BIP32 生成主私钥与链码。每一级子密钥满足：

$$
I=\operatorname{HMAC\text{-}SHA512}(c_{\mathrm{par}},\text{data}),
$$

$$
k_i=
(\operatorname{parse}_{256}(I_L)+k_{\mathrm{par}})
\bmod n,
\qquad c_i=I_R.
$$

沿以太坊常用 BIP44 路径：

```text
m/44'/60'/0'/0/0
```

派生出的私钥为：

```text
0x81c458e9fae445de18385a3379513acc8e191e4c2667c85aa0a52a32ec4e6d55
```

将该私钥乘以 secp256k1 基点得到未压缩公钥，去掉前导 `04` 后计算 Keccak-256，取末 20 字节：

```text
0xaca5872e497f0cc626d1e9ba28baec149315266e
```

与题目给出的地址一致，因此 flag 为：

```text
SEKAI{0x81c458e9fae445de18385a3379513acc8e191e4c2667c85aa0a52a32ec4e6d55}
```

## 方法总结

内存中的命令行参数可能保留已经消失的传输内容和重建命令。面对可疑 `.pyc`，不要相信其表面上的密码提示或固定错误输出，而应直接检查字节码的数据流。本题还需要区分“恢复助记词”和“验证目标账户”：只有按明确的 BIP44 路径派生私钥，并重新计算以太坊地址与题目地址一致，证据链才闭合。
