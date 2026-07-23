# No. 352

## 题目简述

题目给出一张看似普通的森林图片，并提示两个口令来源：

- 口令 1 是全国图鉴编号 352 的宝可梦名称小写；
- 口令 2 位于描述中的连写字符串。

编号 352 是 Kecleon，因此第一层口令为 `kecleon`。图片主题也呼应 Kecleon 的隐身能力。

![用于承载两层 Steghide 隐藏文件的森林图片](UMDCTF2023-no-352-wp/kecleon-carrier.jpg)

仓库 README 把附件写成 PNG，但实际公开文件是 JPEG；Steghide 支持这里的 JPEG 载体，应以文件真实格式为准。

## 解题过程

先检查并提取第一层：

```bash
steghide info hide-n-seek.jpg
steghide extract -sf hide-n-seek.jpg -p kecleon
```

第一层输出本身还是一个 Steghide 载体。题目描述开头给出的连写文本
`timetogofindwhatkecleonishiding` 就是第二层口令：

```bash
steghide info first-output
steghide extract \
  -sf first-output \
  -p timetogofindwhatkecleonishiding
```

第二次提取获得 flag 文件：

```text
UMDCTF{KECLE0NNNNN}
```

两个口令不是需要爆破的未知量：题面分别用图鉴编号和描述原文给出。正确做法是记录每层输出的真实文件类型，再对新载体重复检查。

## 方法总结

- 核心技巧：使用题面提供的两个语义口令，连续解开两层 Steghide 嵌入。
- 识别信号：图片题同时给出明确的两组 password 提示，通常意味着嵌套载荷，而不是同一层的候选口令。
- 复用要点：不要仅按扩展名判断载体；每次提取后重新执行 `file` 或 `steghide info`，并保留层级关系。
