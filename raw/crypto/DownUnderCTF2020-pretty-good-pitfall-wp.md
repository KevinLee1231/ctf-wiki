# DownUnderCTF 2020 - Pretty Good Pitfall

## 题目简述

题目给出二进制文件 `flag.txt.gpg`，并利用 “PGP/GPG 看起来很随机” 诱导选手把它当作密文。实际上该文件只包含一条 OpenPGP signed message：正文仍以明文 literal-data packet 存储，后面附加签名。签名提供完整性与来源认证，不提供机密性。

## 解题过程

先查看 packet 结构，而不是立即寻找私钥：

```bash
gpg --list-packets flag.txt.gpg
```

如果是加密消息，会看到 public-key encrypted session key 或 symmetric-key encrypted data packet；本题则能看到 one-pass signature、literal data 和 signature packet，说明内容可直接提取。

让 GnuPG 处理该 signed file：

```bash
gpg --output flag.txt --decrypt flag.txt.gpg
cat flag.txt
```

GnuPG 会提示缺少签名者公钥，因此无法验证签名是否可信，但这不妨碍读取 literal data：

```text
gpg: Can't check signature: No public key
DUCTF{S1GN1NG_A1NT_3NCRYPT10N}
```

缺公钥意味着“无法确认是谁签的”，不是“正文无法解密”；本题根本没有加密层。

## 方法总结

- 核心技巧：通过 OpenPGP packet 类型区分签名、加密与签名后加密，直接提取 signed message 的 literal data。
- 识别信号：`.gpg` 扩展名和高熵二进制外观不能证明内容已加密；`gpg --list-packets` 的 packet 语义才是判断依据。
- 复用要点：签名解决 authenticity/integrity，只有 encryption 才解决 confidentiality；“无法验证签名”和“无法读取正文”是两个独立状态。
