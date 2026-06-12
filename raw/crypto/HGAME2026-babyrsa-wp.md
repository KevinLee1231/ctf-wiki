# babyRSA

## 题目简述

题目附件是一个小模数 RSA 脚本。脚本随机生成长度 30 到 40 的 flag 主体，字符集为数字、大小写字母、下划线和 `@`，格式为 `VIDAR{...}`；但 `p`、`q` 只有 120 bit，`n` 明显小于完整 flag 转成的整数。题目同时输出 `c`、`p`、`q`。

```python
from Crypto.Util.number import *
from random import *
import string

k = randint(30, 40)
alphabet = string.digits + string.ascii_letters + "_@"
flag = b"VIDAR{" + "".join(choice(alphabet) for _ in range(k)).encode() + b"}"
p = getPrime(120)
q = getPrime(120)
n = p * q
e = 65537
m = bytes_to_long(flag)
c = pow(m, e, n)
print(f"c = {c}")
print(f"p = {p}")
print(f"q = {q}")
```

关键题目信息不是密文数字本身，而是“长格式明文直接取整数后在小模数下加密”，因此公开的 `c` 实际只约束 `m mod n`。

## 解题过程

由于 `m > n`，加密只保留了 `m mod n` 的信息。题目又公开了 `p,q`，所以可以先计算 `n`，再利用 `c = m^e mod n` 得到目标同余。已知前缀 `VIDAR{`、后缀 `}`、长度范围和字符集后，把未知主体按 base256 展开，构造格恢复满足同余的候选字节。原题解还参考了[这篇格恢复思路](https://tangcuxiaojikuai.xyz/post/94c7e291.html)。

```sage
from Crypto.Util.number import *
from itertools import *
import string

e = 65537
p = 722243413239346736518453990676052563
q = 777452004761824304315754169245494387
n = p * q

c = 451420045234442273941376910979916645887835448913611695130061067762180161
newc = c
prefix = b"VIDAR{"
suffix = b"}"

for le in range(37, 46):
    length = le - len(prefix) - len(suffix)
    # remove known prefix and suffix
    c -= 256^(len(suffix) + length) * bytes_to_long(prefix)
    c -= bytes_to_long(suffix)
    c = c * inverse(256, n) % n
    L = Matrix(ZZ, length+2, length+2)
    for i in range(length):
        L[i, i] = 1
        L[i, -1] = 256^i
        c -= 256^i*44
        c -= 256^i*40
    L[-2, -2] = 1
    L[-2, -1] = -c
    L[-1, -1] = n
    L[:, -1:] *= n
    res = L.BKZ()
    for i in res[:-1]:
        flag = ""
        if all(-44 <= j <= 44 for j in i[:-2]):
            if i[-2] == 1:
                for j in i[:-2][::-1]:
                    flag += chr(44 + 40 + j)
            elif i[-2] == -1:
                for j in i[:-2][::-1]:
                    flag += chr(44 + 40 - j)
        if flag:
            print(flag)
    c = newc
#Congr@tulations_you_re4lly_konw_RS4
```

脚本输出的是 flag 主体，补回题目格式即可。这里的关键不是分解 RSA，因为 `p,q` 已经公开，而是意识到小模数导致完整明文只剩同余约束，必须结合格式和字符集恢复原文。

## 方法总结

当 RSA 题出现 `m > n` 或明文直接转整数加密时，要把问题看成模 n 的信息恢复，而不是普通 RSA 解密。若有格式、长度和字符集约束，可构造格把未知字节限制在小范围内恢复。
