# shufflebox

## 题目简述

附件的 `shufflebox.py` 生成一次长度为 16 的随机排列 `PERM`，随后对每一行都执行：

```python
return ''.join(s[PERM[p]] for p in range(16))
```

同一个置换复用于所有行。`output_censored.txt` 给出两组已知明文/密文对，以及未知明文对应的密文 `owuwspdgrtejiiud`。因此这不是需要猜词的文本题，而是恢复一个固定的 16 位置换。

## 解题过程

对输出位置 $p$，有 $o[p]=s[\mathrm{PERM}[p]]$。第一组探针把位置分成 `a`、`b`、`c`、`d` 四类，第二组 `abcd` 重复四次，把每一类再细分；两组字符所在的位置集合取交集即可唯一定位每一个输出位置所引用的输入位置。

官方 `solve/solve.py` 反过来按原文位置收集其可能的输出位置，`possibilities[i]` 最终应只留下一个下标：

```python
possibilities = [set(range(16)) for _ in range(16)]

for s, o in [
    ("aaaabbbbccccdddd", "ccaccdabdbdbbada"),
    ("abcdabcdabcdabcd", "bcaadbdcdbcdacab"),
]:
    for i in range(16):
        candidates = {p for p, ch in enumerate(o) if ch == s[i]}
        possibilities[i] &= candidates

assert all(len(x) == 1 for x in possibilities)
inverse_perm = [next(iter(x)) for x in possibilities]
```

`inverse_perm[i]` 正是满足 $\mathrm{PERM}[inverse\_perm[i]]=i$ 的输出位置，所以原文第 $i$ 个字符为 `ct[inverse_perm[i]]`。代入第三行得到 `udiditgjwowsuper`，最终为：

```text
DUCTF{udiditgjwowsuper}
```

无需重建随机数种子，也无需枚举 $16!$ 个排列。

## 方法总结

固定位置换一旦复用，选择具有不同位置标签的已知明文就能直接恢复它。这里两条四值探针提供 $4\times4=16$ 种位置标签，刚好覆盖全部位置。密码设计上，置换若承担保密作用必须由密钥和每条消息的独立随机性保护；仅对字符洗牌且跨消息复用没有语义安全性。
