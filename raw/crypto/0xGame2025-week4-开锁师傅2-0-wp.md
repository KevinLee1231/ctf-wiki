# 开锁师傅2.0

## 题目简述

附件 `attachment.zip` 使用传统 ZipCrypto 加密，包含 38 个极短的编号文本、一个 1088 字节的 `VIP_file` 和一个 38 字节的 `flag.txt`。ZIP 中央目录即使在文件加密时也会公开每个文件的未压缩长度和 CRC-32。对 1 字节、3 字节文件枚举原文并匹配 CRC，即可拼出 `VIP_file` 的开头明文；随后利用 ZipCrypto 已知明文攻击恢复内部密钥，解密 `flag.txt`。

## 解题过程

### 利用短文件的 CRC-32

CRC-32 只是线性校验码，不具备抗原像恢复能力。对于长度为 1 的文件，候选空间只有 $2^8$；若已知长度为 3 的文件承载单个 UTF-8 中文字符或全角标点，则只需枚举合法的三字节 UTF-8 序列，候选数量约为 $16\times64\times64$，远小于暴力遍历全部 $2^{24}$ 个三字节串。

下面的脚本从 ZIP 中读取文件名、长度和 CRC，同时正确处理“不同文件内容相同、CRC 重复”的情况，再按数字文件名排序拼接：

```python
import zlib
from collections import defaultdict
from pathlib import Path
from zipfile import ZipFile

zip_path = Path("attachment.zip")
targets = defaultdict(list)

with ZipFile(zip_path) as archive:
    for info in archive.infolist():
        if info.file_size in (1, 3):
            targets[info.CRC].append(info.filename)

found_by_crc = {}

# 单字节 ASCII/原始字节。
for value in range(256):
    data = bytes([value])
    crc = zlib.crc32(data) & 0xFFFFFFFF
    if crc in targets:
        found_by_crc[crc] = data

# 合法的三字节 UTF-8 字符。
for first in range(0xE0, 0xF0):
    for second in range(0x80, 0xC0):
        for third in range(0x80, 0xC0):
            data = bytes((first, second, third))
            try:
                data.decode("utf-8")
            except UnicodeDecodeError:
                continue

            crc = zlib.crc32(data) & 0xFFFFFFFF
            if crc in targets:
                found_by_crc[crc] = data

found = {
    name: found_by_crc[crc]
    for crc, names in targets.items()
    if crc in found_by_crc
    for name in names
}

message = b"".join(found[f"{index}.txt"] for index in range(1, 39))
print(message.decode("utf-8"))
```

在仓库原附件上运行得到：

```text
好像有个很重要的文件,就记住了前面的内容:welcome_to_0xGame
```

### 对 ZipCrypto 执行已知明文攻击

`VIP_file` 的开头正是 `welcome_to_0xGame`。把这 18 字节保存为无换行的 `plain`，使用 `bkcrack` 对同一 ZIP 条目的开头执行已知明文攻击：

```bash
printf %s 'welcome_to_0xGame' > plain
bkcrack -C attachment.zip -c VIP_file -p plain
```

PDF 中记录的恢复结果为：

```text
Keys: b4f76b3f 94ffc8f4 b85ceb2b
```

ZipCrypto 的三个 32 位内部 key 对同一归档内的各加密条目通用，因此无需恢复原密码，可以直接解密 `flag.txt`：

```bash
bkcrack -C attachment.zip \
  -k b4f76b3f 94ffc8f4 b85ceb2b \
  -c flag.txt -d flag.txt
```

对仓库原始 ZIP 复核后，`VIP_file` 解密结果确实以已知明文开头，`flag.txt` 的长度与 CRC 也完全匹配。精确 flag 为：

```text
0xGame{WEII_Y0u_@re_nnaSTer_I0cksm1th}
```

## 方法总结

这道题串联了两个弱点：短消息的 CRC-32 几乎等同于可枚举指纹，而传统 ZipCrypto 在取得足够连续已知明文后可以恢复内部状态。分析加密 ZIP 时，应先检查中央目录暴露的长度、CRC 和压缩方式；大量 1–3 字节条目往往就是恢复明文的入口。获得同归档中任一条目的连续已知明文后，再转向 ZipCrypto 已知明文攻击即可解开其他条目。
