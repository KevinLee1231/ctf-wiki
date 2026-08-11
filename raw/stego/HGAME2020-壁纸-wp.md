# 壁纸

## 题目简述

附件表面是一张壁纸，文件尾部却追加了加密 ZIP。ZIP 注释写着 `Password is picture ID.`，需要根据图片作者与作品信息找到 Pixiv 作品编号；解压后的文本又使用了省略前导零的 Unicode 转义。

## 解题过程

先检查文件签名或运行 `binwalk`。在正常图片数据之后可以找到 ZIP 本地文件头：

```text
50 4b 03 04
```

从该偏移切出 ZIP 后读取注释，得到提示：

```text
Password is picture ID.
```

图片文件名包含作者“純白可憐”，据此检索作者作品可定位到 Pixiv 作品 `76953815`。这条外部信息在解题中的实际作用只有一个：作品编号就是压缩包密码，因此无需再依赖原网页是否仍可访问。

用密码 `76953815` 解压后，`flag.txt` 内容为：

```text
\u68\u67\u61\u6d\u65\u7b\u44\u6f\u5f\u79\u30\u75\u5f\u4b\u6e\u4f\u57\u5f\u75\u4e\u69\u43\u30\u64\u33\u3f\u7d
```

标准 `\u` 转义要求后跟四位十六进制数，而题目只保留了每个 ASCII 字节的两位十六进制数。给每组补上前导 `00` 后再解码：

```python
import re

raw = r"\u68\u67\u61\u6d\u65\u7b\u44\u6f\u5f\u79\u30\u75\u5f\u4b\u6e\u4f\u57\u5f\u75\u4e\u69\u43\u30\u64\u33\u3f\u7d"
fixed = re.sub(
    r"\\u([0-9a-fA-F]{2})(?![0-9a-fA-F])",
    r"\\u00\1",
    raw,
)
print(fixed.encode().decode("unicode_escape"))
```

得到：

```text
hgame{Do_y0u_KnOW_uNiC0d3?}
```

## 方法总结

- 核心技巧：从图片尾部雕取追加 ZIP，利用压缩包注释把作品 ID 转化为密码，再修复非标准 Unicode 转义。
- 识别信号：图片末尾出现 `PK\x03\x04`，ZIP 注释指向图片元数据、作者或作品编号。
- 复用要点：检索得到的外部线索必须落实为可验证的具体值；处理 `\u` 时要先确认题目给的是码点还是逐字节十六进制。
