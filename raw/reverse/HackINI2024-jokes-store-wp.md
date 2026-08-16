# HackINI2024 jokes store

## 题目简述

题目提供一个未剥离符号的 64 位 Go 程序。普通菜单只能查看或提交笑话，“special joke” 功能要求输入形如四组 4 字符的 VIP 票据。程序会分别计算四段的 MD5，并与内置哈希比较；全部通过后才读取环境变量 `FLAG`。

## 解题过程

从 `main.HearSpecialJoke` 可以恢复票据格式：

```text
^[0-9A-Z]{4}-[0-9A-Z]{4}-[0-9A-Z]{4}-[0-9A-Z]{4}$
```

`main.CheckTicketValidity` 按 `-` 分割输入，随后 `main.CheckHash` 计算每一段的 MD5，并依次与以下值比较：

```text
efbef50a500a775721a668f0fde8ebfd
28c31a8b4b0a54bf02cdcb08a2cc9d68
1fecc0f2d559d5f440da1820d5399fb2
87ecd4873b986300b07e1d4eff6d95e5
```

每段只有 $36^4=1,679,616$ 种可能，而且四段彼此独立。可以对每个目标哈希枚举一次：

```python
from hashlib import md5
from itertools import product

alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
targets = [
    "efbef50a500a775721a668f0fde8ebfd",
    "28c31a8b4b0a54bf02cdcb08a2cc9d68",
    "1fecc0f2d559d5f440da1820d5399fb2",
    "87ecd4873b986300b07e1d4eff6d95e5",
]

def recover(target):
    for chars in product(alphabet, repeat=4):
        candidate = "".join(chars)
        if md5(candidate.encode()).hexdigest() == target:
            return candidate
    raise ValueError("no preimage in the declared alphabet")

ticket = "-".join(recover(target) for target in targets)
print(ticket)
```

恢复出的票据为：

```text
D5B2-KTSX-ALXK-L3UL
```

重新计算四段 MD5 与二进制中的四个目标完全一致。提交票据后，程序读取 `.env` 中的 `FLAG` 并输出：

```text
shellmates{w3Lc0m3_t0_tH3_w0rLD_0f_g0L4nG}
```

## 方法总结

Go 二进制即使静态链接、体积较大，只要未剥离符号，仍可从 `main.*` 函数快速定位业务逻辑。将 16 字符票据拆成四个独立的 4 字符 MD5 原像，把一次不可行的 $36^{16}$ 搜索降为四次 $36^4$ 搜索，是本题的关键。最终还应把求得的每段重新哈希，形成可验证闭环。
