# notRC4

## 题目简述

题目给出 RC4 密文以及生成完全部密钥流后的置换状态 $S_k$，但不给密钥。解密并不需要逆向 KSA 的密钥：只要把 PRGA 逐轮倒推到其初始状态 $S_0$，再从 $i=j=0$ 正向生成同长度密钥流即可。

## 解题过程

RC4 的一轮 PRGA 为：

```text
i = (i + 1) mod 256
j = (j + S[i]) mod 256
swap(S[i], S[j])
K = S[(S[i] + S[j]) mod 256]
```

若密文长为 $k$，最终 $i_k=k\bmod256$ 已知，而 $j_k$ 未知。枚举 $j_k\in[0,255]$，每轮先撤销交换，再计算 $j_{t-1}=j_t-S_{t-1}[i_t]$ 和 $i_{t-1}=i_t-1$。只有最终回到 $j_0=0$ 的状态才是候选：

```python
def prga(state, length):
    state = state.copy()
    stream = []
    i = j = 0
    for _ in range(length):
        i = (i + 1) % 256
        j = (j + state[i]) % 256
        state[i], state[j] = state[j], state[i]
        stream.append(state[(state[i] + state[j]) % 256])
    return bytes(stream)

def reverse_prga(rounds, final_state):
    candidates = []
    final_i = rounds % 256

    for final_j in range(256):
        state = final_state.copy()
        i, j = final_i, final_j

        for _ in range(rounds):
            state[i], state[j] = state[j], state[i]
            j = (j - state[i]) % 256
            i = (i - 1) % 256

        if i == 0 and j == 0:
            candidates.append(state)
    return candidates

def decrypt(final_state, ciphertext):
    for initial_state in reverse_prga(len(ciphertext), final_state):
        stream = prga(initial_state, len(ciphertext))
        message = bytes(a ^ b for a, b in zip(ciphertext, stream))
        if message.startswith(b"hgame{") and message.endswith(b"}"):
            return message
    raise ValueError("没有通过已知格式校验的候选状态")
```

将附件中的 256 字节最终状态和密文填入后，得到：

```text
hgame{0o00Oo00_~reVeRSe_tHE~prGA_0F-Rc4-+0OoOOOOo}
```

由于已知明文末字节为 `}`，还可由最后一个密文字节推得末轮密钥流字节，用它筛掉大部分 $j_k$；枚举 256 个值本身已经足够快。

## 方法总结

- 核心技巧：逆转 PRGA 状态更新，而不是尝试恢复未知 RC4 密钥。
- 识别信号：若题目给出的是运行若干轮后的完整 $S$，则 KSA 的输出状态仍可能通过可逆 PRGA 回退得到。
- 复用要点：逆序必须先撤销 swap，再更新 $j$ 和 $i$；最后用已知前后缀验证候选，避免把错误状态当作唯一解。

> 公开 Crypto 复盘补足了原 PDF 未展示的最终明文；正文已完整保留 PRGA 逆算法。参考：[HGame 2020 Crypto 题解](https://blog.soreatu.com/posts/writeup-for-crypto-problems-in-hgame-2020/)。
