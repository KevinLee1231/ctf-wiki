# DroidedChungus

## 题目简述

原题要求分析一个 Android 游戏并走出迷宫，但仓库中的 APK 与数据库外链已经失效，仅保留服务端源码。现有证据仍足以复现服务协议：客户端每一步都要提交正确方向，并回答由 16 位 challenge 派生的字母排列。

## 解题过程

服务端把 challenge 与固定秘密 `57346` 异或，再送入三个移位异或组成的 32 位 PRNG：

```python
import numpy as np
import string

def lfsr(seed):
    while True:
        seed = np.int32(seed ^ np.int32(seed >> 7))
        seed = np.int32(seed ^ np.int32(seed << 9))
        seed = np.int32(seed ^ np.int32(seed >> 13))
        yield seed

def knuth(seed):
    letters = list(string.ascii_lowercase)
    generator = lfsr(seed)
    for index in range(len(letters) - 1, 0, -1):
        target = next(generator) % 26
        letters[index], letters[target] = letters[target], letters[index]
    return "".join(letters)
```

收到 challenge 后计算 `knuth(challenge ^ 57346)` 作为 `chall_resp`。移动方向则按仓库中 `solution` 数组提交，其中 `1/2/3/4` 分别表示 `up/down/left/right`。完成整条固定路线后，服务返回：

```text
RZT=gR7Nd(Luhq0XLWO5F>_x~axisdeE
```

这是 Python Base85：

```python
import base64

encoded = b"RZT=gR7Nd(Luhq0XLWO5F>_x~axisdeE"
print(base64.b85decode(encoded).decode())
```

解码得到：

```text
UMDCTF-{Chu4gus_1s_Pr0ud}
```

## 方法总结

本题按现存仓库只能走服务端源码复原路线，不能假装完成了已经缺失的 APK 迷宫逆向。服务协议的关键是准确复刻带 32 位溢出的移位异或和打乱过程，再按服务端保存的方向序列推进，最后解 Base85。
