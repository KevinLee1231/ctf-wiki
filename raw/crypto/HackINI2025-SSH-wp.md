# SSH

## 题目简述

题目名中的 SSH 指的是 “Super Secure Hashing”。服务的“签名”接口会计算 `H(user_message + FLAG)`，但 `H` 不是对整条消息做一次 SHA-256，而是每 40 字节独立哈希并返回摘要列表：

```python
def H(message):
    return [
        sha256(message[i:i + 40]).hexdigest()
        for i in range(0, len(message), 40)
    ]
```

攻击者能控制 flag 前面的字节，并看到每个固定大小块的裸 SHA-256，因此可以把未知 flag 字节逐个挤到首块末尾，用本地字典比较恢复。

## 解题过程

仓库官方 solver 已知 flag 总长为 40。假设当前已恢复前缀长度为 $r$，向签名接口发送

$$
\texttt{"A"}^{39-r}.
$$

服务端首个 40 字节块正好是

$$
\texttt{"A"}^{39-r}\parallel
\text{FLAG}[0:r+1],
$$

也就是填充、已知前缀和一个未知字符。取得这个块的目标摘要后，本地枚举可打印字符 $c$，计算

$$
\operatorname{SHA256}
(\texttt{"A"}^{39-r}\parallel\text{known}\parallel c).
$$

摘要相等时即可确定下一个字符。每轮重新调整填充长度，直到恢复 40 字节：

```python
from hashlib import sha256
import string

FLAG_LEN = 40
CHARSET = string.printable

def recover(get_first_hash):
    known = b""
    while len(known) < FLAG_LEN:
        padding = b"A" * (FLAG_LEN - 1 - len(known))
        target = get_first_hash(padding)

        for ch in CHARSET.encode():
            block = padding + known + bytes([ch])
            if sha256(block).hexdigest() == target:
                known += bytes([ch])
                break
        else:
            raise RuntimeError("no byte matched")
    return known
```

`get_first_hash` 只需选择菜单中的签名功能、发送 payload，并解析返回列表的第一个十六进制摘要。这里不需要碰撞 SHA-256，也不是长度扩展攻击；漏洞来自服务把秘密附在可控前缀后，又暴露独立分块摘要。

恢复结果为：

```text
shellmates{W3Lcom3-to-H4ckINI-2k25-CTF!}
```

## 方法总结

- 核心技巧：控制秘密前缀前的填充，使每轮只有一个未知字节落入可本地枚举的固定长度哈希块。
- 识别信号：服务返回“每块一个摘要”的列表、块之间没有链式状态或随机盐，并计算 `attacker_input || secret` 时，应考虑逐字节字典攻击。
- 复用要点：攻击依赖已知块长和可预测的秘密长度；它利用的是低熵单字节候选和独立分块，不应误写成攻破 SHA-256 原像安全性。
