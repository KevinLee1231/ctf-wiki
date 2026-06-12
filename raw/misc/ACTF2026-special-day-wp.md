# special day

## 题目简述

题目附件只有简单文本/压缩包线索，目标是从文件内容中按题目提示恢复母亲节相关 flag。

## 解题过程

附件 `1.zip` 解压后得到目录 `1/`，其中包含一个文件 `1.txt`。
读取文件内容：

```text
SGFwcHkgTW90aGVyJ3MgRGF5LCBNb20h
```

该字符串仅包含大小写字母、数字和 `+`、`/`、`=` 等 Base64 常见字符，长度为 32，符合 Base64 编码特征，尝试进行 Base64 解码。

解码结果为：

```text
Happy Mother's Day, Mom!
```

题干给出了 flag 的构造规则：

```text
Use _ to join the words, remove punctuation, and wrap it with ACTF{}.
```

因此对解码出的句子进行处理：

1. 原句：`Happy Mother's Day, Mom!`
2. 去除所有标点符号：`Happy Mothers Day Mom`
3. 使用下划线 `_` 连接各单词：`Happy_Mothers_Day_Mom`
4. 用 `ACTF{}` 包裹得到最终 flag。

验证脚本：

```python
import base64
import string

with open("1/1.txt", "r", encoding="utf-8") as f:
    data = f.read().strip()

decoded = base64.b64decode(data).decode("utf-8")
print(decoded)

translator = str.maketrans("", "", string.punctuation)
clean_text = decoded.translate(translator)
flag_body = "_".join(clean_text.split())
flag = f"ACTF{{{flag_body}}}"
print(flag)
```

运行脚本输出 `Happy Mother's Day, Mom!` 和对应 flag，说明日期/文本恢复逻辑正确。

最终得到：

```text
ACTF{Happy_Mothers_Day_Mom}
```

## 方法总结

- 核心技巧：按附件文本的编码或简单变换恢复明文。
- 识别信号：Misc 题只有极小附件且题名强提示日期/节日时，应先检查字符串、编码和日期主题。
- 复用要点：简单题也要在 WP 中说明输入文件形态和变换方式，避免只留 flag。
