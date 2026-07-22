# MAnaGerfAker

## 题目简述

题目要求提交一个经过 Base64 编码的 APK。服务端使用修改版 `apksigner.jar` 验证它必须同时通过 APK Signature Scheme v2 与 v3，随后导出验签过程中实际读取到的每一份 X.509 证书。每份导出证书都必须恰好长 `0x33b` 字节，并满足指定的 32 位滚动哈希。

哈希逻辑来自旧版 KernelSU 对 Manager APK 的识别代码：它从 APK Signing Block 中定位证书字节，以有符号字节参与

$$
h_0=1,\qquad h_{i+1}=31h_i+\operatorname{signed}(b_i)\pmod{2^{32}}.
$$

服务端目标值为 `868400328`。真正的矛盾是：证书必须保持密码学验签有效，但原始字节又要被改成指定长度和指定弱哈希。

## 解题过程

### 找到“同语义、不同编码”的空间

[Android 的 v3 格式说明](https://source.android.com/docs/security/features/apksigning/v3)指出，v3 签名块与 v2 结构相近，`signed data` 中包含长度前缀的 X.509 证书序列；v3 还加入 SDK 范围和 `proof-of-rotation`，使新旧签名证书能够形成受签名保护的轮换链。标准格式要求证书为 ASN.1 DER，但本题所用 `apksig` 的解析路径比标准编码更宽松。

在 [`X509CertificateUtils`](https://android.googlesource.com/platform/tools/apksig/+/refs/tags/android-13.0.0_r59/src/main/java/com/android/apksig/internal/util/X509CertificateUtils.java) 中，输入若带 PEM 头尾，解析器会收集 Base64 主体，并跳过 `Character.isWhitespace` 认可的字符；PEM 尾部之后连续的空白也会被忽略。因此，可以让两段不同的原始字节被解析、重编码为同一份 DER 证书。

[Java 8 的 `Character.isWhitespace`](https://docs.oracle.com/javase/8/docs/api/java/lang/Character.html#isWhitespace-char-)不仅接受空格、制表、换行等字符，还接受 `U+001C` 至 `U+001F` 四个分隔控制字符。于是短 PEM 证书后面可以追加由 `0x1c`、`0x1d`、`0x1e`、`0x1f` 和普通空格组成的可控后缀；这些字节改变原始证书哈希，却不改变 X.509 语义。

题目提供的签名工具还做了两项定向修改：

1. 签名端先导出原始 DER 证书；若目录中存在伪造 PEM，就验证该 PEM 重新编码后的 DER 与原证书完全相同，再把伪造的原始编码写入 APK Signing Block。
2. 验证端导出所有被解析的证书原始字节，便于服务端对每份证书重新计算弱哈希。

因此攻击目标不是伪造 RSA/ECDSA 签名，而是为同一公钥证书构造另一种解析等价的字节表示。

### 将 BKDR 哈希约束写成格问题

原始 ECC PEM 证书只有 667 字节，小于目标长度 827 字节，留下 160 字节后缀空间。固定原证书和大部分尾部空格后，哈希对剩余字节是线性的。若第 $i$ 个可调字节相对基准值 `0x1d` 的改变量为 $x_i$，其贡献权重就是相应的 $31^k\bmod 2^{32}$，需要满足

$$
\sum_i x_i31^{k_i}equiv h_{\text{target}}-h_{\text{base}}\pmod{2^{32}}.
$$

官方 solver 以 `0x1d` 为基准，将短系数 $x_i$、目标差值和模数 $2^{32}$ 一起嵌入整数格，并用 $B=2^{48}$ 放大同余坐标。LLL 找到末两维分别为 `0` 和 $B$ 的短向量后，前面的分量就是各后缀字节的调整量。把它们加回 `0x1d` 后仍落在 Java 接受的控制空白范围内，同时滚动哈希恰好命中目标。

关键约束可以概括为：

```python
hash_mod = 1 << 32
h = 1
for byte in certificate:
    signed = byte if byte < 0x80 else byte - 256
    h = (31 * h + signed) % hash_mod

assert len(certificate) == 0x33b
assert h == 868400328
```

最后使用修改版 `apksigner.jar` 将伪造 PEM 编码写入测试 APK。服务端确认 v2、v3 均验签成功，并确认所有导出证书均满足长度与哈希条件后返回：

```text
ACTF{9982a1ff-ef42-4cfc-8daa-940d9a275f0e}
```

旧版 KernelSU 的具体风险点可以从其 [`apk_sign.c`](https://github.com/tiann/KernelSU/blob/0856b718defc558b0f0e6dfa423ea8f510a09c44/kernel/apk_sign.c#L111) 看出：代码只比较证书长度和 `hash = 31 * hash + signed_byte` 的结果，而不比较规范化证书或强摘要。该问题后来以 [CVE-2023-5521](https://huntr.com/bounties/d438eff7-4e24-45e0-bc75-d3a5b3ab2ea1/) 修复；这条链接用于保留漏洞来源，解题所需的实现条件已经在正文中给出。

## 方法总结

- 核心技巧：利用 X.509 宽松解析产生“解析结果相同、原始编码不同”的证书，再用 LLL 求解 32 位线性滚动哈希的受限后缀碰撞。
- 识别信号：安全边界一侧按结构解析并规范化对象，另一侧却对原始字节做弱哈希时，应检查表示层歧义是否能绕过身份绑定。
- 复用要点：必须同时满足解析等价、长度固定、字符集合受限和模 $2^{32}$ 哈希命中；仅找到普通哈希碰撞或仅让证书可解析都不足以通过验证。
