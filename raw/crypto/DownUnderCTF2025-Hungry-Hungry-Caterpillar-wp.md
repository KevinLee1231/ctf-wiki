# hungry-hungry-caterpillar

## 题目简述

程序把原 flag 追加六倍长度的随机数据，再生成同长度一次性 keystream。随后分别取扩展明文的步长切片 `flag[::s]`（$s=1,2,\ldots,7$），却都从同一个 keystream 的第 0 字节开始异或。不同切片因此复用了相同密钥位置，可从已知字符沿乘法索引传播明文。

## 解题过程

记扩展明文为 $F$、keystream 为 $K$，第 $s$ 行输出为 $C_s$。第 $j$ 个输出字节满足：

$$
C_s[j]=F[sj]\oplus K[j].
$$

第一行满足 $C_1[j]=F[j]\oplus K[j]$，消去 $K[j]$ 可得：

$$
F[sj]=F[j]\oplus C_1[j]\oplus C_s[j].
$$

题目断言 `flag[1] == ord("U")`，所以从索引 1 的已知字符 `U` 出发，可以用 $s\in\{2,3,5,7\}$ 反复传播，恢复所有只含质因子 $2,3,5,7$ 的索引，即 7-smooth 索引。扩展数据总长度是原 flag 的 7 倍，因此原 flag 长度由第一行密文字节数除以 7 得到，只需恢复这一前缀。

官方 solver 对尚未覆盖的较大质数索引做少量 crib dragging：依据格式 `DUCTF{[a-z_]*}` 和已恢复上下文猜一个字符，再继续用同一关系传播。核心过程可写为：

```python
def propagate(known, ciphertexts, flag_len):
    changed = True
    while changed:
        changed = False
        for j in range(flag_len):
            if known[j] is None:
                continue
            for s in range(2, 8):
                idx = s * j
                if idx < flag_len and known[idx] is None:
                    known[idx] = known[j] ^ ciphertexts[0][j] ^ ciphertexts[s - 1][j]
                    changed = True
```

按官方 solver 给出的质数位置猜测并传播后，恢复结果为：

```text
DUCTF{the_hungry_little_p_smooth_caterpillar_wow_an_allegory_for_life}
```

源码中的 `flag.txt` 与该结果一致，可作为静态交叉验证。

## 方法总结

- 核心技巧：消去被重复使用的 keystream 字节，把切片索引关系转成明文字符传播图。
- 识别信号：多份密文使用同一个 keystream 起点，但每份明文是不同步长的同一数据切片；题目又提供一个已知字符。
- 复用要点：先精确写出 `zip(flag[::s], keystream)` 的下标关系，避免把它误当成普通多次 one-time pad。smooth number 只覆盖由给定步长质因子生成的索引，剩余质数索引仍需格式约束或可靠 crib，不能凭空猜完整 flag。
