# L3akCTF 2025 Ghost In The Dark Writeup

## 题目简述

附件是密码为 `L3akCTF` 的 ZIP，其中包含一个约 100 MB 的 Windows 磁盘镜像 `GhostInTheDark.001`。NTFS 分区里只剩勒索说明和若干 `.enc` 文件；负责解密并执行 payload 的 PowerShell loader 已经自删除。

本题要从 NTFS 元数据中恢复已删除脚本，逐层取得两组 AES-CBC 参数，最终解密 `flag.enc`。

## 解题过程

### 识别并挂载磁盘镜像

```bash
unzip -P L3akCTF Ghost.zip
file GhostInTheDark.001
mmls GhostInTheDark.001
```

`mmls` 显示每扇区 512 字节，NTFS 分区从扇区 128 开始，因此字节偏移为：

$$
128\times512=65536.
$$

可以只读挂载：

```bash
mkdir ghost
sudo mount -o ro,loop,offset=65536 GhostInTheDark.001 ghost
```

分区中能看到：

```text
flag.enc
payload.enc
ransom_note.txt
RIP_PuppyJaws.enc
trip_itinerary.enc
System Volume Information
```

勒索说明提示“key 可能仍在黑暗中，script 也消失了”，指向已删除文件恢复。

### 从 MFT 恢复 `loader.ps1`

用 The Sleuth Kit 递归列出分区文件：

```bash
fls -r -o 128 GhostInTheDark.001
```

输出中有：

```text
-/r * 40-128-1: loader.ps1
```

`*` 表示目录项已删除，但 MFT 记录及数据仍可恢复。`icat` 使用元数据地址前的 inode 编号：

```bash
icat -o 128 GhostInTheDark.001 40 > loader_recovered.ps1
```

脚本中的第一层参数为：

```text
Key: 0123456789abcdef
IV:  abcdef9876543210
Mode: AES-CBC
Padding: PKCS7
Input: payload.enc 的 Base64 解码结果
```

loader 解密 `payload.enc`、把明文作为 PowerShell 执行，最后删除自身。

### 解密第二阶段 PowerShell

```python
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b"0123456789abcdef"
iv = b"abcdef9876543210"
encoded = open("payload.enc", "rb").read()
ct = base64.b64decode(encoded)
payload = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
open("payload_recovered.ps1", "wb").write(payload)
```

恢复出的 payload 会读取系统盘上的明文 flag，使用另一组参数加密后写入移动盘，并加密其他文件、删除原文：

```text
Key: m4yb3w3d0nt3x1st
IV:  l1f31sf0rl1v1ng!
Mode: AES-CBC
Padding: PKCS7
Input: flag.enc 的原始二进制
```

这里的 `flag.enc` 不是 Base64 文本，不能重复套用第一层的解码步骤。

### 解密 flag

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b"m4yb3w3d0nt3x1st"
iv = b"l1f31sf0rl1v1ng!"
ct = open("flag.enc", "rb").read()

flag = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
print(flag.decode())
```

得到：

```text
L3AK{d3let3d_but_n0t_f0rg0tt3n}
```

官方 PDF 还说明了题目制作边界：作者创建了固定大小 100 MB 的 VHD，初始化为 NTFS 并分配为 `L:`。明文 flag 只从 `C:` 读取，从未写入证据盘；两层脚本都自删除。这样可以避免直接 `strings` 镜像拿 flag，同时保留足够的 MFT 删除记录供取证恢复。

## 方法总结

文件“已删除”只意味着目录项或分配状态改变，不代表 MFT 记录和数据立即消失。先用 `mmls` 得到正确分区偏移，再用 `fls` 找出带 `*` 的删除项、用 `icat` 按 inode 恢复，是这题的主线。恢复脚本后还要区分两层输入格式：`payload.enc` 是 Base64 包装的 AES-CBC 密文，`flag.enc` 则是原始密文字节。
