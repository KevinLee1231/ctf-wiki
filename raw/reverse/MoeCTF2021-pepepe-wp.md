# PEPEPE

## 题目简述

题目提供一个被破坏的 32 位 PE 和一个经重复密钥加密的文件。第一层破坏清空了 DOS 头中的 `e_lfanew`，导致加载器找不到 NT 头；修复程序后可以看到加密关系：

$$
c_i=\neg(p_i\oplus k_{i\bmod |k|})
=p_i\oplus k_{i\bmod |k|}\oplus\mathtt{0xff}.
$$

该关系覆盖生成程序循环处理的前 `file_len - 1` 个字节；样本末字节未参与变换。

目标文件原本也是 PE，因此 DOS stub 中稳定出现的字符串可作为已知明文，恢复重复密钥并解密完整文件。

## 解题过程

### 1. 修复 `e_lfanew`

PE 文件以 DOS 头开始，偏移 `0x3c` 的四字节小端字段 `e_lfanew` 指向 `PE\0\0` 签名所在的 NT 头。本样本的 NT 头位于文件偏移 `0x80`，所以应把 `0x3c` 到 `0x3f` 修复为：

```text
80 00 00 00
```

修复后文件能够被 Windows 加载，也可以在 32 位 IDA 中正常识别节表和函数。

### 2. 用 DOS stub 做已知明文攻击

加密文件中与标准 DOS stub 文本对应的密文字节如下。由

$$
k_i=p_i\oplus c_i\oplus\mathtt{0xff}
$$

可直接恢复该位置的密钥流：

```python
known_plain = b"This program cannot be run in DOS mode"
known_cipher = bytes([
    222, 182, 228, 233, 169, 234, 255, 249, 253, 255,
    233, 251, 179, 240, 242, 248, 250, 245, 242, 176,
    232, 187, 173, 232, 252, 244, 173, 255, 244, 173,
    204, 217, 192, 179, 254, 249, 240, 255,
])

stream = bytes(
    plain ^ cipher ^ 0xff
    for plain, cipher in zip(known_plain, known_cipher)
)
print(stream)
```

输出呈现明显周期：

```text
u!reverierwilllikeyou!reverierwilllike
```

结合周期边界旋转后，最短重复密钥为：

```text
reverierwilllikeyou!
```

### 3. 解密完整 PE

```python
from pathlib import Path

key = b"reverierwilllikeyou!"
encrypted = Path("file").read_bytes()

decrypted_body = bytes(
    value ^ 0xff ^ key[index % len(key)]
    for index, value in enumerate(encrypted[:-1])
)

# 生成程序的循环上界是 file_len - 1，最后一字节没有经过变换。
# 当前样本的该字节本来就是 0，直接保留。
plain = decrypted_body + encrypted[-1:]

assert plain.startswith(b"MZ")
Path("origin.exe").write_bytes(plain)
```

运行恢复出的程序或查看其中的明文字符串，可以得到：

```text
moectf{P3_Structur3_1s_r3ally_fUnnY!}
```

## 方法总结

- 核心技巧：修复 DOS 头的 `e_lfanew`，再利用 PE DOS stub 的稳定文本对重复异或密钥做已知明文恢复。
- 识别信号：文件以 `MZ` 开头但工具找不到 `PE\0\0` 时，应优先检查 `0x3c` 指针；密文目标仍是 PE 时，可尝试标准 stub 字符串和固定文件头。
- 复用要点：恢复出的密钥流可能从周期中间开始，要先求最短周期并校准相位；完整解密后用 `MZ`、NT 头和程序行为交叉验证。
