# aluminum-isopropoxide

## 题目简述

题目给出同一份 34 字节 flag 的三个加密备份，以及自制流加密程序 `main.cpp`。程序看似有密钥调度、状态数组和 S 盒，但其更新式

```cpp
i = (i * j) % 256;
j = (i + S[j]) % 256;
K = S[i] & S[j];
out[ctr] ^= S_box[K];
```

从 `i = 0` 开始后永远令 `i = 0`。密钥流只沿 `j = S[j]` 的退化轨道生成，字节分布严重偏斜；多个独立备份又重复加密相同的受限字符明文，因此可以做统计恢复。

## 解题过程

明文字符只会出现在 `abcdefghijklmnopqrstuvwxyz_{}` 中。官方 `alipo.py` 保存了一张针对该退化生成器得到的 256 字节经验频率表：对每个密文字节，按密钥流字节的频率从高到低枚举，并只保留异或后落入允许字符集的候选。

三个备份在相同位置对应同一个明文字节。把各备份给出的候选排名累加，就能为每个位置建立字符代价；再用 10000 词词表、`tjctf{` 前缀、下划线分隔和右花括号约束搜索低总代价的单词序列。其核心可概括为：

```python
allowed = "abcdefghijklmnopqrstuvwxyz_{}"

def candidates(cipher_byte, frequency_rank):
    out = []
    for rank, key_byte in enumerate(frequency_rank):
        plain = cipher_byte ^ key_byte
        if chr(plain) in allowed:
            out.append((chr(plain), rank))
    return out

# 对三个备份逐位置合并 rank，再用词表搜索最小代价字符串。
```

脚本的逐位置最低代价串仍有少数字符误判，词典搜索则把正文缩小为一组短语候选，其中包含：

```text
flag_under_mountain_of_dust
```

它满足固定长度、`tjctf{...}` 格式和自然语言结构，并在三个备份的对应位置都落入高排名候选；提交验证得到完整 flag：

```text
tjctf{flag_under_mountain_of_dust}
```

## 方法总结

- 自制 PRGA 最危险的问题不是“看起来不像 RC4”，而是状态变量是否真的覆盖预期状态空间；这里乘法更新让 `i` 永久锁死为 0。
- 同一明文的多个独立密文可以显著增强统计信号；已知 flag 字符集和自然语言词边界则把逐字节候选变成可搜索约束。
- 频率攻击可能保留多个近似短语，输出应以格式、词典、多份密文一致性和最终提交复核，不能把单个最高频候选直接当作数学唯一解。
