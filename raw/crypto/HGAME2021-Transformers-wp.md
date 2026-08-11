# Transformers

## 题目简述

附件包含一份单独密文，以及被碎纸机打乱顺序的多组明文、密文语料。加密只对英文字母做固定的单表替换，标点、数字和字符位置不变。虽然文件配对关系被打乱，但大语料中的字符频率仍基本保持，可以利用两边的频率排名重建替换表，再解开目标文本。

## 解题过程

分别拼接 `data/enc` 与 `data/ori` 下的所有文件，统计字母频率，并把密文侧和明文侧按出现次数从高到低排列。对应排名组合成映射表后，对目标文件执行替换：

```python
from collections import Counter
from pathlib import Path


def load_corpus(directory: str) -> str:
    return "".join(
        path.read_text(encoding="utf-8")
        for path in sorted(Path(directory).iterdir())
        if path.is_file()
    )


encrypted = load_corpus("data/enc")
original = load_corpus("data/ori")

enc_order = "".join(
    char for char, _ in Counter(encrypted).most_common() if char.isalpha()
)
ori_order = "".join(
    char for char, _ in Counter(original).most_common() if char.isalpha()
)

translation = str.maketrans(enc_order, ori_order)
target = Path("data/Transformer.txt").read_text(encoding="utf-8")
print(target.translate(translation))
```

频率映射给出的目标密文为：

```text
Tqh ufso mnfcyh eaikauh kdkoht qpk aiud zkhc xpkkranc uayfi kfieh 2003,
oqh xpkkranc fk "qypth{hp5d_s0n_szi^3ic&qh11a_}",
Dai'o sanyho oa pcc oqh dhpn po oqh hic.
```

结合空格、标点和上下文修正少数频率接近字母后，完整明文是：

```text
The lift bridge console system has only used password login since 2003,
the password is "hgame{ea5y_f0r_fun^3nd&he11o_}",
Don't forget to add the year at the end.
```

最后一句要求在右花括号前补上比赛年份 `2021`，得到：

```text
hgame{ea5y_f0r_fun^3nd&he11o_2021}
```

官方 PDF 没有记录目标明文和最终补年结果，这两项通过 [MiaoTony 的同期复盘](https://miaotony.xyz/2021/02/07/CTF_2021HgameWeek1/) 补齐；所需内容已经写入正文，外链只用于来源核对。

## 方法总结

单表替换保留字符频率，但短文本的频率波动很大；本题额外提供的大量对应语料正是为了稳定估计映射。频率排名只能生成初始字典，遇到次数相近的字母仍需利用英文单词、标点位置和固定 flag 格式人工校正，并继续执行明文中的语义指令。
