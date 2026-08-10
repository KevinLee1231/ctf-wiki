# ezPRNG

## 题目简述

题目给出 4 段、每段 1000 bit 的 LFSR 输出。寄存器宽度为 32 bit，反馈位由以下抽头异或产生：

$$
\operatorname{nextbit}
=R_1\oplus R_5\oplus R_8\oplus R_{13}\oplus R_{18}
\oplus R_{22}\oplus R_{25}\oplus R_{29}\oplus R_{32}.
$$

目标是利用已知输出反向恢复每段序列对应的 32 bit 初始状态，再把四个状态拼成 flag 内容。

## 解题过程

LFSR 正向移位时，旧状态中除被移出的一个 bit 外，其余 31 bit 都会保留在新状态中。反馈函数又给出了“旧状态抽头异或 = 新输出 bit”的线性关系，因此只要把其余已知抽头移到等式另一侧，就能逐位求出被移出的未知 bit。

把当前 32 bit 窗口记为 `window`，每轮取流末尾对应的已知输出 bit，与位置 `1、4、8、11、15、20、25、28` 的已知项异或，即得到前一状态的新首位。连续逆推 32 次便恢复一个完整状态：

```python
def recover_word(stream: str) -> int:
    if len(stream) < 32 or set(stream) - {"0", "1"}:
        raise ValueError("stream 必须是至少 32 bit 的二进制字符串")

    window = stream[:32]
    recovered = []

    for index in range(32):
        # 最前面的 ? 是本轮待恢复的旧状态 bit。
        known = "?" + window[:31]
        bit = int(stream[-1 - index])
        for offset in (1, 4, 8, 11, 15, 20, 25, 28):
            bit ^= int(known[-offset])

        recovered.append(str(bit))
        window = str(bit) + window[:31]

    return int("".join(recovered)[::-1], 2)


# outputs 替换为题目给出的 4 段 1000-bit 输出。
outputs = [output_1, output_2, output_3, output_4]
flag_body = "".join(f"{recover_word(stream):08x}" for stream in outputs)
print(f"VIDAR{{{flag_body}}}")
```

4 段千位输出属于题目原始数据，正文不重复堆放。对 PDF 中的原始序列运行上述逆推，四个状态分别为：

```text
ad156a23
b288a362
b7e0f239
cff7525d
```

最终得到：

```text
VIDAR{ad156a23b288a362b7e0f239cff7525d}
```

## 方法总结

- 核心技巧：利用 LFSR 的线性反馈关系反向逐位恢复状态。
- 识别信号：已知寄存器宽度、反馈抽头和足够长的连续输出，且反馈只由 GF(2) 上的异或构成。
- 复用要点：要统一寄存器的位序、移位方向和输出位定义；同一组抽头在相反下标约定下会得到完全不同的恢复结果，恢复后应重新正向生成输出进行校验。
