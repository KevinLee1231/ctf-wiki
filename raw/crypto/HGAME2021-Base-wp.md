# Base全家福

## 题目简述

题目给出一串多层 Base 编码，依次使用 Base64、Base32 和 Base16。三种编码可以通过字符集和填充特征区分：Base64 包含大小写字母、数字、`+`、`/` 且通常最多两个 `=`；Base32 只使用大写字母和数字 `2` 至 `7`，填充可能超过两个 `=`；Base16 只包含十六进制字符 `0-9A-F`。

## 解题过程

观察每一层解码后的字符集，按 Base64、Base32、Base16 的顺序处理：

```python
import base64

cipher = (
    b"R1k0RE1OWldHRTNFSU5SVkc1QkRLTlpXR1VaVENOUlRHTVlETVJCV0dVMlVNTlpV"
    b"R01ZREtSUlVIQTJET01aVUdSQ0RHTVpWSVlaVEVNWlFHTVpER01KWElRPT09PT09"
)
stage1 = base64.b64decode(cipher)
stage2 = base64.b32decode(stage1)
plain = base64.b16decode(stage2)
print(plain.decode())
```

三层解码后的最终输出为：

```text
hgame{We1c0me_t0_HG4M3_2021}
```

## 方法总结

识别 Base 编码时不要只依赖末尾的 `=`，应同时检查整个字符表和长度约束。每完成一层都先查看输出是可读文本、另一层编码还是原始二进制，再决定下一步；这能避免把 Base32 字符串误当成受限字符集的 Base64。
