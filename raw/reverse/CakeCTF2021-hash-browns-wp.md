# CakeCTF2021 Hash browns

## 题目简述

程序要求输入 74 字节 flag，并把每一对字符拆开校验。第 $2i$ 个字符计算 MD5，和 `hash1[i]` 的前 10 个十六进制字符比较；第 $2i+1$ 个字符计算 SHA-256，却放到由扩展欧几里得算法得到的置换位置。

哈希算法本身没有被攻破。每次哈希的输入只有单个字节，因此直接枚举 0 到 255 即可反查。

## 解题过程

### 还原奇数字符的索引置换

当前公开仓库中的两张表各有 37 项，因此 $p=37$。程序调用扩展欧几里得算法求系数 $x,y$：

$$
ix+py=\gcd(i,p).
$$

对 $1\le i<37$，$p$ 为素数，所以 $x\bmod37=i^{-1}\bmod37$。生成器把第 $2i+1$ 个字符的 SHA-256 前缀存入 `hash2[x]`；$i=0$ 时单独落在索引 0。

需要注意：仓库中有一份按 31 项表和旧二进制偏移读取的早期脚本，与当前发布的 37 项构建不一致。应从当前二进制的字符串区或随仓库发布的 `chall.c` 提取两张实际表，并以表长作为 $p$。

### 单字节枚举恢复两条字符流

```python
from hashlib import md5, sha256

def egcd(a, b):
    if a == 0:
        return b, 0, 1
    g, y, x = egcd(b % a, a)
    return g, x - (b // a) * y, y

p = len(hash1)  # 当前构建为 37
flag = bytearray()

for i in range(p):
    _, inv, _ = egcd(i, p)
    inv %= p

    even = next(
        c for c in range(256)
        if md5(bytes([c])).hexdigest()[:10] == hash1[i]
    )
    odd = next(
        c for c in range(256)
        if sha256(bytes([c])).hexdigest()[:10] == hash2[inv]
    )
    flag.extend((even, odd))

print(flag.decode())
```

每个位置至多尝试 256 个候选。虽然只比较 40 位哈希前缀，但在如此小的候选域中出现歧义的概率极低；若出现多个候选，再结合 flag 格式即可筛选。

恢复结果为：

```text
CakeCTF{(^o^)==(-p-)~~(=_=)~~~POTATOOOO~~~(^@^)++(-_-)**(^o-)_486512778b4}
```

## 方法总结

- 强哈希并不能弥补极小输入空间：单字节原像只能提供 8 位未知量。
- 逆向校验器时要同时还原“比较什么”和“放在哪个索引”，本题的扩展欧几里得只是一个模逆置换。
- 解题脚本中的硬编码表长和文件偏移可能随构建变化，表长应从当前附件动态确认。
