# scramble

## 题目简述

题目给出一份行序被打乱的 Python 程序，只有前三行 `import random`、固定种子和 `random.seed` 的顺序已知。目标先是依据缩进、变量定义和控制流依赖恢复脚本，再运行其确定性变换得到 flag。

## 解题过程

按语法和数据流整理时，可以利用以下约束：

- `recur` 必须先处理长度为 1 的基例，再递归处理偶数位和奇数位切片；
- `decrypt` 先由输入长度确定 `n`，生成随机数组 `l`，再依次计算 `l2`、`l3` 和与常量 `l4` 异或的结果；
- `main` 最后构造 31 个 1 并调用 `decrypt`。

恢复出的核心顺序为：

```python
import random

random.seed(1000)

def recur(values):
    if len(values) == 1:
        assert values[0] > 0
        return values[0]
    return recur(values[::2]) / recur(values[1::2])

def decrypt(inp):
    n = len(inp)
    values = [random.randint(6, 420) for _ in range(n)]
    target = [70, 123, 100, 53, 123, 58, 105, 109, 2, 108, 116, 21, 67,
              69, 238, 47, 102, 110, 114, 84, 83, 68, 113, 72, 112, 54,
              121, 104, 103, 41, 124]
    stage2 = [0] * n
    for i in range(1, n):
        stage2[i] = (values[i] * 5 + (stage2[i] + n) * values[i]) % values[i]
        stage2[i] += inp[i]
    stage2[0] += int(recur(stage2[1:]) * 50)
    stage3 = [0] * n
    stage3[0] = stage2[0] % 256
    for i in range(1, n):
        mixed = values[i] & (
            stage3[i - 1] + (stage3[i - 1] * values[i]) % 256
        )
        stage3[i] = (stage2[i] ^ (mixed // 2)) % 256
    return "".join(chr(target[i] ^ stage3[i]) for i in range(n))

print(decrypt([1] * 31))
```

运行恢复后的脚本得到：

```text
tjctf{unshuffling_scripts_xdfj}
```

## 方法总结

- 行打乱题首先是语法恢复：函数头、缩进块、基例、变量定义先后和 `return` 位置通常能确定大部分结构。
- 固定随机种子使恢复后的脚本完全可复现，可用输出长度与 flag 格式验证排序。
- 不需要逆推出每个算术表达式的闭式；先恢复真实执行顺序，再运行和逐层检查中间变量更高效。
