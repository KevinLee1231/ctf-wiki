# BYUCTF 2022 - Take Me Out to the Ball Game

## 题目简述

题面以棒球、满垒和 grand slam 暗示 “base”，给出一串仅由 `0-9A-S` 组成的字符：

```text
16DLF592KN4KKI9BNJHCM179GML30PB32235B04MDM145GJD04N9NFL
```

## 解题过程

字符集合一共可表示 29 个数值，因此第一层是 Base29。使用字母表 `0123456789ABCDEFGHIJKLMNOPQRS`，即数字仍取 0～9、`A` 到 `S` 依次取 10～28。把整串按 Base29 解释并转回字节后，结果仍不是直接可读文本，而是一串 ASCII 十六进制字符。

可以用任意支持自定义进制的工具完成第一层，也可自行实现：逐字符令累计值满足 `$n \leftarrow 29n+v(c)$`，最后把大整数转成大端字节串。第一层结果再执行 hex 解码：

```python
plain = bytes.fromhex(base29_decoded_text).decode()
```

最终得到：

```text
byuctf{4ll_th3_b4s3s_4r3_10ad3d!}
```

## 方法总结

识别未知 Base 编码时，应先统计字符集和最大符号，而不是只尝试 Base32、Base64 等常见格式。第一层输出呈现纯十六进制特征，说明还需继续做一次表示层解码。
