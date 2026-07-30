# NepCTF2026 true_ezgame Writeup

## 题目简述

服务要求连续赢下 40 轮石头剪刀布。庄家先公布随机数 `r` 和对其出拳的 RSA 风格 commitment，再等待玩家出拳。看似 commitment 应隐藏庄家选择，但掩码同时与只有三个候选值之一异或；由于 `r` 和 RSA 公钥均公开，可以逐一验证三个候选并在提交前确定庄家出拳。

## 解题过程

程序编号为：

```text
rock = 0
scissors = 1
paper = 2
```

消息映射为：

$$
H(i,r)=\operatorname{SHA512}(\operatorname{byte}(i)\parallel r).
$$

commitment 生成过程为：

```python
mask = random_invertible_integer()
c1 = pow(mask, e, n)
c2 = H(dealer, r) ^ mask
```

服务输出 `(c1, c2)`、`r` 以及公共参数 `(n,e)`。对每个候选 $i\in\{0,1,2\}$，假设它是庄家出拳并恢复：

$$
m_i=c_2\oplus H(i,r).
$$

只有正确候选满足：

$$
m_i^e\bmod n=c_1.
$$

核心代码如下：

```python
from hashlib import sha512

MOVES = ("rock", "scissors", "paper")

def H(i, r):
    return int.from_bytes(
        sha512(bytes([i]) + r).digest(), "big"
    )

def recover_dealer(c1, c2, r, n, e):
    for dealer in range(3):
        mask = c2 ^ H(dealer, r)
        if pow(mask, e, n) == c1:
            return dealer
    raise ValueError("no valid commitment candidate")

def winning_move(dealer):
    player = (dealer + 2) % 3
    return MOVES[player]
```

服务中的胜负判断是：

```python
dealer == (player + 1) % 3
```

所以恢复庄家编号后发送 `(dealer + 2) % 3` 对应的字符串。连接开始还有一个 `sha256(XXX + suffix)` 的 3 字符 PoW，枚举给定字符表即可。逐轮解析 `r` 和 commitment、验证候选并发送克制出拳，连续完成 40 轮后服务返回 flag。

## 方法总结

问题不在 RSA 本身，而在承诺值空间过小且候选可公开验证。对只有三个可能消息的 commitment，如果观察者能从每个候选恢复掩码并用公开关系验证，隐藏性就完全失效。设计承诺方案时，消息必须经过具备随机化和抗字典攻击性质的编码，不能简单地与可验证掩码异或。
