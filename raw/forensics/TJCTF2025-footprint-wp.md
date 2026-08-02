# footprint

## 题目简述

题目目录中的普通文件都已被删除，只留下 macOS Finder 创建的 `.DS_Store`。被删文件的内容为空，flag 被拆成两段后分别进行 URL-safe Base64 编码，并直接用编码结果作为文件名。因此恢复目标不是文件内容，而是 `.DS_Store` 的目录记录中残留的旧文件名。

## 解题过程

`.DS_Store` 是带 B-tree 结构的目录元数据文件。即使目录项对应的文件随后被删除，其中记录过的名称仍可能保留。使用附件自带的解析器列出记录：

```bash
python3 parse.py files/.DS_Store > output.txt
```

输出中大部分名称是随机字节编码后形成的噪声。对每行补齐 Base64 padding，做 URL-safe Base64 解码，只保留可打印 UTF-8，即可找出两条有意义记录：

```python
import base64
import string

with open("output.txt", "r", encoding="utf-8") as f:
    for line in f:
        name = line.strip()
        if not name:
            continue
        try:
            raw = base64.urlsafe_b64decode(name + "=" * (-len(name) % 4))
            text = raw.decode("utf-8").rstrip()
        except (ValueError, UnicodeDecodeError):
            continue
        if text and all(ch in string.printable for ch in text):
            print(name, "->", text)
```

关键输出为：

```text
dGpjdGZ7ZHNfc3RvcmVfIA -> tjctf{ds_store_
aXNfdXNlZnVsP30gICAgIA -> is_useful?}
```

按文件名内容拼接得到：

```text
tjctf{ds_store_is_useful?}
```

## 方法总结

- 核心技巧：从 `.DS_Store` 的残留目录项恢复已删除文件名，再解码文件名承载的数据。
- 识别信号：macOS 目录只剩 `.DS_Store`、题面强调“文件名”而非内容、可疑名称符合无 padding 的 URL-safe Base64 字符集。
- 复用要点：删除文件不等于清除所有目录元数据；解码大量候选时应以 UTF-8、可打印率和 flag 前缀筛选，不要手工逐条猜测。
