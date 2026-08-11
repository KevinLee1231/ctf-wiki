# 夺宝大冒险2

## 题目简述

服务端用已知 40 位反馈掩码的 LFSR，每轮连续产生 4 位随机数，玩家在 100 轮中答对至少 80 次即可获得 flag。答错时服务端会公开正确值。虽然可以用 Z3、Berlekamp–Massey 或 GF(2) 线性方程恢复初始种子，但本题状态更新为左移并把输出位补到最低位；观察 40 个连续输出位后，当前状态的低 40 位就完全由这 40 位组成，可直接从第 11 轮开始预测。

## 解题过程

生成器源码如下：

```python
class LXFIQNN:
    def __init__(self, init, mask, length):
        self.init = init
        self.mask = mask
        self.lengthmask = 2 ** (length + 1) - 1

    def next(self):
        nextdata = (self.init << 1) & self.lengthmask
        selected = self.init & self.mask & self.lengthmask
        output = 0
        while selected != 0:
            output ^= selected & 1
            selected >>= 1
        nextdata ^= output
        self.init = nextdata
        return output

    def random(self, nbit):
        output = 0
        for _ in range(nbit):
            output <<= 1
            output |= self.next()
        return output
```

固定参数为：

```python
mask = 0b1011001010001010000100001000111011110101
length = 40
```

记第 $i$ 次反馈位为 $o_i$。忽略不参与反馈的额外高位，低 40 位更新满足：

$$
S_{i+1}=((S_i\ll1)\mathbin{\&}(2^{40}-1))\mathbin{|}o_i.
$$

每轮都会丢掉一个最高位并从最低位补入一个输出位，所以经过 40 次更新，原状态的 40 位全部被挤出，当前低 40 位依次就是 $o_0,o_1,\ldots,o_{39}$。服务端的 `random(4)` 按高位在前拼接 4 个输出，因此只要收集前 10 轮并把每个数补足 4 位，就能得到当前状态。

前 10 轮始终猜 0：猜错时响应会泄露正确的 `secret`；恰好猜中时正确值本来就是 0，同样已知。之后用这 40 位初始化同一生成器，预测剩余 90 轮，最终得分至少为 90。

```python
import re

from pwn import *


MASK = 0b1011001010001010000100001000111011110101
LENGTH = 40


class LXFIQNN:
    def __init__(self, init, mask, length):
        self.init = init
        self.mask = mask
        self.lengthmask = 2 ** (length + 1) - 1

    def next(self):
        nextdata = (self.init << 1) & self.lengthmask
        selected = self.init & self.mask & self.lengthmask
        output = selected.bit_count() & 1
        self.init = nextdata ^ output
        return output

    def random(self, nbit):
        value = 0
        for _ in range(nbit):
            value = (value << 1) | self.next()
        return value


io = remote("target.example", 30607)
observed = []

for _ in range(10):
    io.sendlineafter(b"guess: ", b"0")
    reply = io.recvline().strip()
    if reply == b"Right":
        observed.append(0)
    else:
        match = re.search(rb"secret is (\d+)", reply)
        if not match:
            raise RuntimeError(f"无法解析响应: {reply!r}")
        observed.append(int(match.group(1)))

bits = "".join(f"{value:04b}" for value in observed)
assert len(bits) == 40
current_state = int(bits, 2)
predictor = LXFIQNN(current_state, MASK, LENGTH)

for _ in range(10, 100):
    guess = predictor.random(4)
    io.sendlineafter(b"guess: ", str(guess).encode())
    reply = io.recvline().strip()
    if reply != b"Right":
        raise RuntimeError(f"预测失配: guess={guess}, reply={reply!r}")

print(io.recvall().decode(errors="replace"))
```

公开复盘中保存的最终结果为：

```text
hgame{lfsr_121a111y^use-in&crypto}
```

题目源码、原始的“回推初始种子”脚本和该 flag 可在 [huangx607087 的 HGAME Div3/4 题解](https://huangx607087.online/2021/02/28/HgameDiv3/) 交叉核验；这里采用的是官方 PDF 同样提到、但更直接的“40 位输出就是当前状态”方法。

## 方法总结

已知掩码时，LFSR 输出对状态是 GF(2) 上的线性关系，但不一定非要解方程。本题更新方向会把反馈位依次写入状态，观察窗口长度恰好等于 40 后，旧种子已经不再影响参与反馈的低 40 位。利用“猜错泄露正确值”的协议，只牺牲前 10 轮就能预测后 90 轮，超过 80 分门槛；拼接每个 4 位输出时必须保留前导零，否则状态位序会错乱。
