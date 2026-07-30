# Onchain_Checkin

## 题目简述

题目给出一个经过删减的 Solana Anchor 工程。附件中的 `Anchor.toml` 指明目标位于 Devnet，`lib.rs` 则留下地址：

```text
SUCTF2Q25DnchainCheckin11111111111111111111
```

本题不是要求补全、编译并部署项目，而是要求从已经存在的链上交易中找回三段 Base58 数据。源码把三处载体分别伪装成日志、指令名和账户公钥。

## 解题过程

先在 [Solscan Devnet 账户页](https://solscan.io/account/SUCTF2Q25DnchainCheckin11111111111111111111?cluster=devnet) 查询附件留下的地址。该地址关联两笔交易；逐笔查看后，目标交易为：

```text
21hrX9ekAihzk5M1fE7EdagACu1LGJj8j4bBbU12oNc26nxdGpXknyXTXhUzG9ukuEgnPV2h5M5Yb57geD4vgjnk
```

对应的 [交易详情页](https://solscan.io/tx/21hrX9ekAihzk5M1fE7EdagACu1LGJj8j4bBbU12oNc26nxdGpXknyXTXhUzG9ukuEgnPV2h5M5Yb57geD4vgjnk?cluster=devnet) 给出完整的指令、账户和日志。结合仓库中未删减的出题源码，可以确定三段数据的来源：

1. `checkin()` 通过 `msg!()` 写入日志 `3LDqJJCHwDBGQP9Zn5MSx`；
2. Anchor 指令 `you_have_found` 在浏览器中显示为 `YouHaveFound`；
3. `account3` 的公钥为 `7Qgd9aqwprLzfS4L9KQFM3mNdG3WpjevNoCoRduXXfPS`，同时被写入 `CheckinState.flag3`。

依次对三段字符串做 Base58 解码，再按上述顺序拼接。下面的脚本不依赖 Solana SDK，只实现必要的 Base58 解码：

```python
ALPHABET = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"


def b58decode(text: str) -> bytes:
    value = 0
    for char in text:
        value = value * 58 + ALPHABET.index(char)
    raw = value.to_bytes((value.bit_length() + 7) // 8, "big")
    return b"\x00" * (len(text) - len(text.lstrip("1"))) + raw


parts = [
    "3LDqJJCHwDBGQP9Zn5MSx",
    "YouHaveFound",
    "7Qgd9aqwprLzfS4L9KQFM3mNdG3WpjevNoCoRduXXfPS",
]
print(b"".join(map(b58decode, parts)).decode())
```

输出为：

```text
SUCTF{Con9ra7s!YouHaveFound_7HE_KEeee3ey_P4rt_0f_Th3_F1ag.}
```

## 方法总结

这是一道链上取证式签到题。附件中的程序地址是检索入口，而不是让选手重新部署的完整项目；决定性步骤是去 Devnet 浏览器查看历史交易，并理解 Anchor 指令名、程序日志和账户公钥都可能承载 Base58 文本。链上数据本身已经给出全部证据，本地源码主要用于确认三段数据的语义和拼接顺序。
