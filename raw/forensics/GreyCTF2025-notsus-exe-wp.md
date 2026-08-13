# notsus.exe

## 题目简述

附件 `files.zip` 中有受传统 ZipCrypto 保护的 `notsus.exe` 与 `flag.txt.yorm`。解题分两层：先利用 PE 文件的已知 DOS Stub 明文恢复 ZIP 内部密钥，再逆向可执行文件所使用的 RC4 逻辑，解开 `.yorm` 文件。

## 解题过程

Windows PE 文件通常包含固定文本 `This program cannot be run in DOS mode.`。题目给出的已知明文还包含其前一个字节 `!`，共 40 字节：

```text
!This program cannot be run in DOS mode.
```

这段明文在未计 ZipCrypto 12 字节加密头时从偏移 `77` 开始。用 `bkcrack` 对同一归档中的 `notsus.exe` 发起已知明文攻击：

```bash
bkcrack -C files.zip -c notsus.exe -p dosstring.txt -o 77
```

本地复现恢复出三个内部密钥：

```text
d1608c35 d11d350a 4bc3da9c
```

随后生成解除密码保护的归档：

```bash
bkcrack -C files.zip \
  -k d1608c35 d11d350a 4bc3da9c \
  -D decrypted.zip
unzip decrypted.zip
```

此时得到 `flag.txt.yorm`，但内容仍由程序自身加密。分析 `notsus.exe` 或仓库中的生成脚本可见，它以固定密钥 `HACKED!` 执行标准 RC4：先用 KSA 打乱 256 字节状态表，再用 PRGA 生成密钥流与文件异或。下面的脚本可完成第二层解密：

```python
from pathlib import Path

key = b"HACKED!"
data = Path("flag.txt.yorm").read_bytes()

s = list(range(256))
j = 0
for i in range(256):
    j = (j + s[i] + key[i % len(key)]) % 256
    s[i], s[j] = s[j], s[i]

i = j = 0
plain = bytearray()
for value in data:
    i = (i + 1) % 256
    j = (j + s[i]) % 256
    s[i], s[j] = s[j], s[i]
    plain.append(value ^ s[(s[i] + s[j]) % 256])

print(plain.decode())
```

输出为：

```text
grey{this_program_cannot_be_run_in_dos_mode_hehe}
```

## 方法总结

ZipCrypto 的安全性不足以抵抗已知明文攻击，而 PE 的固定 DOS Stub 恰好提供了长度充足、位置可预测的明文。恢复 ZIP 内容后仍不能把 `.yorm` 当作最终答案，还要继续分析文件生成逻辑。分清“容器加密”和“载荷加密”两层，逐层验证文件格式与明文结构，能避免在第一层成功后过早停止。
