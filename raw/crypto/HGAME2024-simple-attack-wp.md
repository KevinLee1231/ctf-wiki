# simple_attack

## 题目简述

题目给出一个使用传统 ZipCrypto 加密的 ZIP，以及能够构造同名已知明文文件的条件。利用已知明文攻击可以恢复 ZipCrypto 的三个内部密钥，无需找出原始口令。解密出的 `flag.txt` 是一段 Base64 图片数据，解码图片即可看到 flag。

## 解题过程

### 准备匹配的已知明文

已知明文文件必须与加密 ZIP 中目标成员的原始字节完全一致。可以把该文件单独压成 `plain.zip`，并保证内部路径仍是 `flag.txt`。不同压缩软件可能采用不同方法或元数据，导致 GUI 工具无法匹配；如果题目同时给了官方 `src.zip`，直接将它作为明文归档通常更稳。

### 恢复 ZipCrypto 内部密钥

官方题解使用 ARCHPR 的“明文攻击”。等价地，可以用 `bkcrack` 直接指定密文归档、密文成员、明文归档和明文成员：

```bash
bkcrack -C attachment.zip -c flag.txt -P plain.zip -p flag.txt
```

攻击成功后会输出三项内部密钥。把它们代入下一条命令生成解密归档：

```bash
bkcrack -C attachment.zip -k <key0> <key1> <key2> -D decrypted.zip
```

这里的目标只是恢复可解密数据的内部密钥，不需要继续反推出用户设置的 ZIP 密码。

### 将 Base64 还原为图片

解压 `decrypted.zip` 后读取 `flag.txt`。如果文本带有 `data:image/png;base64,` 前缀，应先去掉逗号之前的部分：

```python
import base64
from pathlib import Path

encoded = Path("flag.txt").read_text(encoding="utf-8").strip()
if encoded.startswith("data:"):
    encoded = encoded.split(",", 1)[1]

Path("flag.png").write_bytes(base64.b64decode(encoded))
```

打开 `flag.png`，图片中的文字为：

```text
hgame{s1mple_attack_for_zip}
```

## 方法总结

- 核心技巧：对传统 ZipCrypto 实施已知明文攻击，恢复内部密钥后解密归档。
- 识别信号：加密 ZIP 采用 ZipCrypto，而题目同时给出、泄露或允许重建其中某个文件的精确明文。
- 复用要点：明文必须字节级一致，归档内路径和压缩方式也会影响工具匹配；攻击成功后保存三项内部密钥即可，不必浪费时间继续恢复口令。
