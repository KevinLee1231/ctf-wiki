# DownUnderCTF 2023 randomly chosen Writeup

## 题目简述

生成脚本用 `random.randrange(0, 1337)` 选择随机种子，然后调用 `random.choices` 从 flag 的字符下标中有放回抽样，输出长度为 flag 长度的 5 倍。种子空间只有 1337 个，因此 Python 伪随机序列可以直接穷举重放。

## 解题过程

设公开字符串长度为 $L$，原 flag 长度就是 $L/5$。对每个候选种子重置 Python 的 `random` 状态，重新生成同样数量的下标；公开字符串第 $i$ 个字符应当属于重放出的第 $i$ 个下标位置。

```python
import random
from pathlib import Path

output = Path("output.txt").read_text().strip()
flag_length = len(output) // 5

for seed in range(1337):
    random.seed(seed)
    positions = random.choices(range(flag_length), k=len(output))
    recovered = [""] * flag_length
    for char, position in zip(output, positions):
        recovered[position] = char

    candidate = "".join(recovered)
    if candidate.startswith("DUCTF{") and candidate.endswith("}"):
        print(seed, candidate)
        break
```

正确种子重放出的字符位置相互一致，得到：

```text
DUCTF{is_r4nd0mn3ss_d3t3rm1n1st1c?_cba67ea78f19bcaefd9068f1a}
```

## 方法总结

伪随机数生成器在种子和调用序列相同的情况下是确定性的。题目虽然进行了大量随机抽样，但种子空间极小，且公开输出长度足以覆盖每个位置；穷举种子并用固定前后缀校验即可可靠恢复 flag。
