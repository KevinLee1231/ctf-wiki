# 惜字如金 2.0

## 题目简述

附件是一份被“惜字如金化”的 Python 程序。变换只作用于连续拉丁字母组成的单词，并按两条规则删除字符：

1. 单词末尾的 `e`/`E`，若前一字符是辅音，则删除该 `e`/`E`；
2. 连续重复的同一辅音（忽略大小写）只保留第一个。

程序中的五个码表字符串现在都长 23，但代码要求原始长度为 24；随后将它们拼接，并按 40 个固定下标取字符生成 flag。目标是为每行恢复恰好一个被删除的字符，并利用 `flag{...}` 的结构约束确定正确码表。

决定性主障碍是逆转一组字符删除规则并恢复编码表，属于表示层编码问题，因此归入 `crypto`。

## 解题过程

### 枚举每行的合法原像

对压缩后字符串中的每个辅音，原串可能在它后面多出：

- 同一辅音的小写或大写形式，对应连续辅音折叠；
- 若该辅音已经位于单词末尾，则还可能多出 `e` 或 `E`。

注意恢复的字符应插在辅音之后。可用下面的函数为每一行生成候选：

```python
CONSONANTS = set("bcdfghjklmnpqrstvwxyzBCDFGHJKLMNPQRSTVWXYZ")
LETTERS = set("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ")

def recover_row(s):
    out = []
    for i, ch in enumerate(s):
        if ch not in CONSONANTS:
            continue

        # 原串可能含两个连续的同一辅音。
        out.append(s[:i + 1] + ch.lower() + s[i + 1:])
        out.append(s[:i + 1] + ch.upper() + s[i + 1:])

        # 原串也可能在词尾辅音后有被删除的 e/E。
        if i == len(s) - 1 or s[i + 1] not in LETTERS:
            out.append(s[:i + 1] + "e" + s[i + 1:])
            out.append(s[:i + 1] + "E" + s[i + 1:])
    return list(dict.fromkeys(out))
```

五行压缩码表为：

```python
rows = [
    "nymeh1niwemflcir}echaet",
    "a3g7}kidgojernoetlsup?h",
    "ulw!f5soadrhwnrsnstnoeq",
    "ct{l-findiehaai{oveatas",
    "ty9kxborszstguyd?!blm-p",
]
```

### 用 flag 结构约束候选

原程序给出的下标属于拼接后的 120 字符码表。下标 `c` 对应第 `c // 24` 行、第 `c % 24` 列。枚举五行候选并按下标取字符，同时要求：

```python
flag.startswith("flag{")
flag.endswith("}")
"}" not in flag[5:-1]
```

可以在做笛卡尔积之前按各行负责的输出位置过滤候选。修正后的候选数从 `[34, 30, 38, 26, 42]` 缩减为 `[8, 12, 7, 6, 23]`，枚举后只有一个不同的输出：

```text
flag{you-ve-r3cover3d-7he-an5w3r-r1ght?}
```

把恢复出的五行拼接后按原程序的 40 个索引取值，结果满足开头 `flag{`、内部无额外 `}`、最后一位为 `}`，构成完整验证链。

## 方法总结

- 核心技巧：为有损字符变换枚举局部原像，再用输出格式和固定索引关系消除歧义。
- 识别信号：程序自身断言码表每行应为 24 字符，但附件中的每行只有 23 字符，且题面精确定义了可删除字符的位置和种类。
- 复用要点：逆向删除规则时要严格确认插入位置；先把全局约束拆成逐行过滤，可显著缩小笛卡尔积。最终不能只凭“像 flag”判断，还应验证所有原程序断言。
