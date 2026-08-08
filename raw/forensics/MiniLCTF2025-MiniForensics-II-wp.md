# MiniForensicsⅡ

## 题目简述

题目延续前一题解密后的 TLS 流量，从中取得使用传统 ZipCrypto 加密的 `lock.zip`。归档里有 PNG 文件，因此可以利用 PNG 固定文件头实施 ZipCrypto 已知明文攻击；解锁归档后得到一个 Base64 编码的 GitHub 仓库地址和一段生成历史提交哈希的脚本。最终证据藏在指定的旧提交中，而不在当前分支文件树里。

## 解题过程

### 用 PNG 固定头攻击 ZipCrypto

PNG 前 16 字节固定为：

```text
89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52
```

将这 16 字节写入已知明文文件，再对归档中的 PNG 条目运行 `bkcrack`：

```python
from pathlib import Path

Path("known-prefix.png").write_bytes(bytes.fromhex(
    "89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52"
))
```

```powershell
bkcrack.exe -C lock.zip -c useless.png -p known-prefix.png
```

官方样本恢复出的三个内部 key 为：

```text
45797e52 f747cc4c 800bd117
```

使用实际输出的 key 生成解锁后的归档：

```powershell
bkcrack.exe -C lock.zip -k 45797e52 f747cc4c 800bd117 -U unlocked.zip temporary-password
```

如果实际附件的 entry 名或 key 不同，应使用 `bkcrack -L lock.zip` 确认条目名，并以本次攻击输出为准，不能机械照抄样本常量。

### 沿 Base64 线索定位旧提交

解锁后的 `breadcrumb.txt` 经 Base64 解码得到公开仓库地址：

```python
import base64
from pathlib import Path

raw = Path("breadcrumb.txt").read_bytes().strip()
print(base64.b64decode(raw).decode())
```

仓库当前分支主要是诱饵文件，但运行 `the_simulation_of_calculating_hash.py` 会给出历史提交对象：

```text
89045a3653af483b6bb390e27c10db16873a60d1
```

直接访问该 [GitHub 历史提交](https://github.com/root-admin-user/what_do_you_wanna_find/commit/89045a3653af483b6bb390e27c10db16873a60d1)，可以看到当时新增、后来从主线消失的 `secret.py`。这一步的关键不是在当前四个可见提交标题中寻找相同前缀，而是把完整 SHA 当作 Git 对象标识直接查询。

### 读懂旧版 secret.py

旧提交中的真实路径为：

```python
import hashlib
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

seed = "miniLCTF_secret_seed_2023"
key = hashlib.sha256(seed.encode()).digest()[:16]

# encrypted_flag 的前 16 字节是随机 IV，剩余部分是 CBC 密文。
iv = encrypted_flag[:16]
ciphertext = encrypted_flag[16:]
plain = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext), 16)
print(f"miniLCTF{{{plain.decode()}}}")
```

同一文件还构造了多层 Base64、zlib、逆序和 hex 诱饵，但 `main()` 最终直接解密本次生成的 `encrypted_flag`。运行该历史版本得到：

```text
miniLCTF{c0ngr4tul4ti0n5_70u'v3_g0t_th3_s3cr3ts}
```

## 方法总结

- 核心技巧：利用 PNG 固定头恢复 ZipCrypto 内部密钥，再用完整 Git commit SHA 查询当前分支不可见的历史文件。
- 识别信号：ZipCrypto 加密归档中存在具有固定文件头的 PNG；解锁后出现 GitHub URL 和 40 位十六进制字符串。
- 复用要点：已知明文必须对应加密条目的真实开头；Git 提交 SHA 是内容寻址对象标识，文件从当前分支删除并不等于旧对象立即不可访问。外链中的 AES-CBC、seed 和诱饵层次已经写入正文，未来无需依赖网页才能理解解法。
