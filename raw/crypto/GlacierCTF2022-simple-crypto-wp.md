# GlacierCTF2022 - Simple Crypto

## 题目简述

附件只有一段由大小写字母、数字等字符组成的长密文。生成器先写入一段随机前缀，然后对 flag 的每个字符执行“写一个真实字符，再写 100 个随机小写字母”，本质上是固定步长的栅栏式抽取题。

## 解题过程

已知 flag 以 `glacierctf{` 开头，可以同时枚举起始偏移和步长。对每个候选 $(i,j)$，依次检查 `ciphertext[i + k*j]` 是否与已知前缀一致；一旦整个前缀吻合，再输出同一等差位置上的全部字符：

```python
known = "glacierctf{"

for offset in range(50):
    for step in range(1, len(ciphertext) // len(known)):
        candidate = ciphertext[offset::step]
        if candidate.startswith(known):
            print(offset, step, candidate)
```

附件的零基偏移为 29，步长为 101：真实字符之间虽然只有 100 个填充字符，但从一个真实字符走到下一个真实字符还要计入自身，所以不是 100。官方说明中的 offset 30 对应一基计数。

直接计算 `ciphertext[29::101]` 得到：

```text
glacierctf{St1ck_3m_uP}
```

## 方法总结

固定周期插入随机噪声不会隐藏原文的等差位置结构。已知格式前缀可同时充当偏移和周期的判别器；分析此类题时应明确零基/一基偏移，并区分“间隔中的噪声数量”与“相邻有效字符的步长”。
