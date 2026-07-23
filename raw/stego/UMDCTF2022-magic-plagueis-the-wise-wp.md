# UMDCTF2022 Magic Plagueis the Wise Writeup

## 题目简述

原题附件包含 4464 个按数字命名、没有扩展名的文件。每个文件的主体都像 PNG，但第一个魔数字节被替换成了一个文本字符。题目把长文本逐字符藏进文件头，再以文件名保存顺序；决定性障碍是隐藏载荷的重组，因此归入 `stego`。

当前仓库未保留原始大附件，但保留了官方恢复脚本、完整明文和 flag，可以据此还原准确流程。

## 解题过程

PNG 的标准文件头以十六进制 `89 50 4e 47 0d 0a 1a 0a` 开始。检查这些无扩展名文件时，会发现除首字节外其余内容符合 PNG 特征，而首字节恰好落在可打印 ASCII 范围。

按数值顺序读取 `1` 到 `4464`，每个文件取第一个字节并拼接：

```python
from pathlib import Path

root = Path("magic-files")
plaintext = bytearray()

for i in range(1, 4465):
    with (root / str(i)).open("rb") as fp:
        plaintext.extend(fp.read(1))

print(plaintext.decode("utf-8"))
```

恢复结果是重复多次的 Darth Plagueis 故事，flag 被夹在连续文本中。搜索比赛前缀即可定位：

```text
UMDCTF{d4r7h_pl46u315_w45_m461c}
```

文件名必须按整数排序；若按字符串排序，会出现 `1, 10, 100, ...`，从而打乱明文。

## 方法总结

魔数异常不一定只是文件损坏。如果大量同构文件都只在同一偏移出现可打印差异，就应考虑把该偏移视为跨文件信道。重组时要同时确认字符来源、文件顺序和终止范围；本题的数字文件名与官方脚本中的 `1..4464` 正好给出完整顺序。
