# rusty vault

## 题目简述

Rust 程序把 32 字节 AES-256-GCM key、12 字节 nonce 和 `result` 常量直接编入二进制。用户输入去掉末尾换行后调用 `Aes256Gcm::encrypt`，只有产生的 `ciphertext || 16-byte tag` 与 `result` 相同才显示成功。没有口令哈希或不可逆校验：密钥已公开在程序常量中，目标是恢复实现与字节序后解密既有密文，故归入 Reverse。

## 解题过程

### 分离 GCM 输出

源码中 `result` 长 60 字节。Rust `Aead::encrypt` 返回 AEAD ciphertext 后接 GCM authentication tag，因此把它分为：

```text
ciphertext = result[:-16]
tag        = result[-16:]
```

key 与 nonce 分别是源中的固定数组。已知 key/nonce 后，不是在攻击 GCM：对 ciphertext 走 GCM 的 CTR 加密流即可恢复候选明文；保留 tag 则可在重新加密时验证完整等式。

### 处理反汇编中的字节序

官方 solver 展示二进制反编译时，常量以 128 位 little-endian 块出现，因此每个从 `xmmword` 抄录的 16 字节块须反转回内存字节序，再与末尾 8 字节、4 字节块拼接。使用正确 key/nonce 对 `ciphertext` 解密，所得文本再交给同一 AES-GCM API 加密，应再现 `result`（包括 tag）。

### 验证

恢复的值为 `DUCTF{enCrypTi0n_I5_NoT_Th3_S@me_as_H@sh1ng}`，与题目配置和生成源码一致。本文未执行 Rust binary 或官方 Python solver；结果来自嵌入 Rust 常量、生成器和 solver 的静态复核。

## 方法总结

- 核心技巧：对称加密把 key、nonce 和密文同时硬编码进程序时，安全性已经失效；认证 tag 并不阻止持钥者恢复明文。
- 识别信号：`Aes256Gcm::new`、固定 nonce、固定结果数组和“加密输入再比较”时，应优先判断是否可直接以反向加密恢复输入。
- 复用要点：区分 GCM ciphertext 与 tag，并留意 SIMD 常量在反汇编中的端序展示；本题分类依据是提取与理解程序常量，不是密码数学攻击。
