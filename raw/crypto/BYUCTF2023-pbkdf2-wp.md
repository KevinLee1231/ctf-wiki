# BYUCTF 2023 - PBKDF2

## 题目简述

服务使用 AES-256 加密 ZIP，已知原始长密码，却明确拒绝该字符串。关键是 ZIP AES 的 PBKDF2-HMAC-SHA1：超过 SHA-1 块大小 64 字节的 HMAC 密钥会先被 SHA-1 压缩成 20 字节摘要，因此长密码与其原始摘要字节导出相同密钥。

## 解题过程

构建镜像时使用的长密码是：

```text
isnt-byuctf-one-of-your-most-favorite-ctfs-even-though-this-is-only-our-second-year-3HF4z
```

其 SHA-1 原始摘要恰好全部可打印：

```text
:h;V4\%T2EV(@~]Y62$8
```

对应十六进制为：

```text
3a683b56345c255432455628407e5d5936322438
```

这不是摘要的十六进制文本，而是 20 个真实摘要字节按 ASCII 解释。服务只按字符串值屏蔽原密码，所以把该等价密码做 URL 编码后传给 `password` 参数即可解压并读出：

```text
byuctf{th4nk_y0u_4rs3n1y_sh4r0g14z0v}
```

仓库中的 `calculate.py` 固定长前缀、枚举短后缀，直到 `hashlib.sha1(...).digest()` 的每个字节都落在可打印 ASCII 范围，从而专门生成了这个可通过 HTTP 传递的实例。

## 方法总结

官方 README 把机制简写成“ZIP 对长密码取 SHA-1”。更准确地说，是 PBKDF2 内部 HMAC 的长密钥规范化造成等价口令。区分 `digest()` 原始字节与 `hexdigest()` 文本是本题最容易出错的地方。
