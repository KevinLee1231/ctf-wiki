# BYUCTF 2023 - Collision

## 题目简述

验证器接收两个 Base64 字符串，要求解码后都以 PNG 文件签名开头、MD5 完全相同，并分别包含两段不同的固定安全宣传文本。每个 Base64 输入最多 10000 字符。

## 解题过程

验证器只检查前八字节 PNG 签名，并不真正解码图片，因此可以使用 [corkami/collisions](https://github.com/corkami/collisions) 提供的 PNG chosen-prefix collision 构造器。仓库已给出两个小载体 `pic1.png`、`pic2.png`，分别放入验证器要求的文本，再执行：

```bash
python3 png.py pic1.png pic2.png
md5sum collision1.png collision2.png
base64 -w0 collision1.png
base64 -w0 collision2.png
```

生成的两个文件内容不同但 MD5 相同，并各自保留对应固定字符串。依次提交两段 Base64 后，服务输出：

```text
byuctf{c0ll1s10n5_4r3_c00l10}
```

## 方法总结

普通碰撞只要求找到任意两条消息，chosen-prefix collision 则允许保留两个不同前缀，更适合满足“不同语义、相同 MD5”的验证器。文件魔数检查不等同于格式有效性验证，但本题真正的决定性原语仍是 MD5 可构造碰撞。
