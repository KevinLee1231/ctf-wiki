# SwiftPasswordManager: LoadMe

## 题目简述

题目给出同一密码管理器保存的 `passwords.spm`，并告知主密码为 `DUCTF2025!`。应用界面的 Load 功能被注释掉，连同 `SPMFileManager.load`、`CryptoHelper.decrypt` 和 `BinaryDecoder` 都未编入可用路径；但 Save 代码、被注释的加载代码以及官方 `solv.py` 足以完整恢复文件格式和解密流程。

因此无需修复 GUI。主障碍是从 Swift 源码重建二进制封装、重复 KDF、AES-GCM 认证解密和 entry 序列化格式。

## 解题过程

### 解析 `.spm` 外层格式并派生密钥

文件以小端整数依次保存：

```text
u32 magic = 0x314D5053  # 文件字节为 SPM1
u16 version
u16 flags
u16 salt_len, salt
u16 nonce_len, nonce
u16 tag_len, tag
u32 ciphertext_len, ciphertext
```

`deriveKey` 不是 PBKDF2：以主密码 UTF-8 字节为初值，固定迭代 `0xaaaa` 次，每次计算：

$$K_{i+1}=\operatorname{SHA256}(K_i\mathbin\Vert salt)$$

最终 32 字节 digest 作为 AES-GCM key。解密时 nonce、ciphertext、tag 分别来自文件，AAD 为 `nil`；需把 ciphertext 与 tag 按 AEAD API 需要的形式组合并验证 tag，不能忽略认证失败。

### 还原明文 entry 序列

认证解密得到的 payload 先是 `u32 entry_count`。每个条目按以下顺序编码：16 字节 UUID、四个 `u32 little-endian length + UTF-8 bytes` 字段（title、username、password、notes），以及两个 `i64 little-endian` Unix 时间戳（created、modified）。

这是 `BinaryEncoder.encode(LoginEntry)` 的直接序列化规则；官方 solver 用同样的 offset 递进读取字段。解密后遍历所有 entry，password 字段中恢复题目要求的 flag。

以下伪代码保留了关键 KDF，不依赖失效 GUI：

```python
key = b"DUCTF2025!"
for _ in range(0xAAAA):
    key = sha256(key + salt).digest()

plaintext = AESGCM(key).decrypt(nonce, ciphertext + tag, None)
entries = parse_u32_count_then_entries(plaintext)
```

### 验证

随附 `solve/solv.py` 与 `FileManager.swift`、`CryptoHelper.swift` 在 magic、字段顺序、`0xAAAA` 次 SHA-256、AES-GCM 以及 entry 解析上完全对应；题目给定主密码是唯一外部输入。本文只审阅了文件、源码和 solver，未执行脚本、安装 cryptography 依赖或修改附件。

## 方法总结

- 核心技巧：当 GUI 加载路径被禁用时，从写入路径和注释掉的 decoder 反向重建格式；保存代码常是最可靠的格式规范。
- 识别信号：自定义容器同时包含 salt、nonce、tag、密文长度时，优先检查 KDF 循环、AEAD AAD 和二进制字段的端序/长度类型。
- 复用要点：认证解密成功后仍要按源码的 UUID、长度前缀、时间戳顺序解析；只把解密文本当作 UTF-8 会丢失结构并导致偏移错误。
