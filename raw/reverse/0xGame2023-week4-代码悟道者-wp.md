# 代码悟道者

## 题目简述

附件是一个 JAR。程序读取用户输入，调用 `customBase64Encode` 编码后与内置密文比较；需要从 Java 字节码中恢复自定义 Base64 字母表并逆向解码。

## 解题过程

反编译 `Main.class`，可以直接定位两个关键常量：

```text
encodedSecret   = MH7HYWrb4p2oYpYtMcEvLTb8Np2jOD2mMoqqYTauLTatYTarMWYvMp7bMdq=
customBase64Map = ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz-_
```

`customBase64Encode` 仍按每 3 字节拆成 4 组 6 bit 索引，只是索引使用了不同的 64 字符表。因此把密文字符从自定义表映射回标准 Base64 表，再正常解码即可：

```python
import base64

encoded = "MH7HYWrb4p2oYpYtMcEvLTb8Np2jOD2mMoqqYTauLTatYTarMWYvMp7bMdq="
custom = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz-_"
standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

translated = encoded.translate(str.maketrans(custom, standard))
print(base64.b64decode(translated).decode())
```

运行结果为：

```text
0xGame{72c672a9-9b77-8703-4a98-97a951f938e2}
```

## 方法总结

本题只是替换了 Base64 的索引字母表，位分组和 `=` 填充规则没有改变。确认算法结构后，建立“自定义表到标准表”的一一映射即可还原明文，无须依赖在线解码工具。
