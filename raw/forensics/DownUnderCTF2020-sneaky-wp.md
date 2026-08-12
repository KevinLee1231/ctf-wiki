# DownUnderCTF 2020 - Sneaky

## 题目简述

题目给出一份约 500 个数据包的 PCAP，混有 DNS、HTTP、HTTPS、ICMP 和 SMB。HTTP/SMB 中出现的 `flag.txt` 与 `tryharder.png` 都是诱饵；真正证据藏在一条异常长的 DNS 查询中。查询载荷解码后依次形成口令 ZIP 和 Vim 加密文件，最终还要利用旧 Vim Blowfish 模式的已知明文弱点恢复 flag。

## 解题过程

### 从 DNS 查询中提取载荷

按协议筛选流量后，HTTPS 内容无法直接读取，而 DNS 查询名通常是明文。异常查询以 `remote.` 开头、以 `.reallylegit.com` 结束，中间各 label 拼接起来像 Base64-URL：字符集中出现 `-`、`_`，同时没有标准 Base64 的 `=` 填充。

可以从 Wireshark 复制完整查询名，或用 `tshark` 导出后处理。解码逻辑如下：

```python
import base64
from pathlib import Path

fqdn = Path("query.txt").read_text().strip()
assert fqdn.startswith("remote.")
assert fqdn.endswith(".reallylegit.com")

payload = fqdn[len("remote."):-len(".reallylegit.com")]
payload = payload.replace(".", "")
payload += "=" * (-len(payload) % 4)

Path("payload.bin").write_bytes(base64.urlsafe_b64decode(payload))
```

`file payload.bin` 将结果识别为 ZIP。该步骤也解释了域名中间的点：DNS label 有长度限制，发送者把同一段 Base64-URL 切成多个 label，并不是对每段分别编码。

### 破解 ZIP 口令

ZIP 受传统口令保护，先用 `zip2john` 转成 John the Ripper 可处理的哈希，再用 `rockyou.txt` 进行字典攻击：

```bash
mv payload.bin payload.zip
zip2john payload.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
john --show zip.hash
```

恢复出的口令为：

```text
narutoshippudengoku
```

解压后得到名为 `flag` 的文件，但其开头不是明文，而是：

```text
VimCrypt~02!
```

### 利用 VimCrypt 已知明文

`VimCrypt~02!` 表示旧 Vim Blowfish 模式。文件前 28 字节由 12 字节算法标识、8 字节 salt 和 8 字节 IV 组成，之后才是密文。该旧模式在每个 8 字节块上重复使用同一段异或 keystream；只要知道第一个明文块，就能推出 keystream 并解出每个块的对应位置。

flag 固定以 `DUCTF{` 开头，因此先用 6 字节已知明文试探：

```python
from pathlib import Path


def xor_bytes(left: bytes, right: bytes) -> bytes:
    return bytes(a ^ b for a, b in zip(left, right))


def recover(path: str, known_first_block: bytes) -> bytes:
    data = Path(path).read_bytes()
    assert data[:12] == b"VimCrypt~02!"

    ciphertext = data[28:]
    blocks = [ciphertext[i:i + 8] for i in range(0, len(ciphertext), 8)]
    keystream = xor_bytes(blocks[0], known_first_block)
    return b"".join(xor_bytes(block, keystream) for block in blocks)


print(recover("flag", b"DUCTF{").decode("ascii", errors="replace"))
```

第一次只能解出每个 8 字节块的前 6 字节，结果呈现为：

```text
DUCTF{y_H4rD_!EE7}
```

从分块位置可以看出已知字符不足造成了间隔，而可见片段很自然地指向 `Try_Harder`。把已知首块扩展为完整的 `DUCTF{Tr` 再运行：

```python
print(recover("flag", b"DUCTF{Tr").decode("ascii", errors="replace"))
```

最终得到：

```text
DUCTF{Try_H4rD3R_!EE7}
```

上述文字与代码已逐页对照原 5 页 PDF 转写；PDF 中的 Wireshark、终端和代码截图均为可可靠转写的文本信息，因此没有机械保留页面截图。

## 方法总结

- 核心技巧：从 DNS label 重组无填充 Base64-URL，破解内层 ZIP，再对旧 VimCrypt Blowfish 模式实施已知明文恢复。
- 识别信号：PCAP 中异常长的 DNS 查询、固定前后缀和高熵 label 常指向 DNS 隧道或数据外带；文件以 `VimCrypt~02!` 开头时要进一步判断旧模式的弱点。
- 复用要点：DNS 中的点可能只是分片边界；Base64-URL 需恢复字符表和填充；已知明文短于分组时只能解出各块相同位置的部分字符，可利用 flag 格式和已恢复词片逐步扩展到完整分组。
