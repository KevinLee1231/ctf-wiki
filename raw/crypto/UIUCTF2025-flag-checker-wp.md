# flag_checker

## 题目简述

程序要求输入 8 个无符号整数。第 $i$ 个输入 $x_i$ 必须满足：

$$
\text{test\_pt}_i^{x_i}\equiv \text{test\_ct}_i\pmod P,
\qquad P=4294967087=0xffffff2f.
$$

通过检查后，程序再计算 `flag_enc[i]^x_i mod P`，把 8 个 32 位结果按本机内存布局当作字符串打印。决定性步骤是分别求 8 个有限域离散对数，而不是爆破输入；虽然题目发布在 Reverse 分类下，这一步属于密码数学主障碍，因此归入 Crypto。

## 解题过程

快速幂函数 `F(b, e, m)` 是标准的 square-and-multiply。`check_input()` 对每个位置独立验证，因此可直接求：

$$
x_i\equiv\log_{\text{test\_pt}_i}(\text{test\_ct}_i)
\pmod{\operatorname{ord}_P(\text{test\_pt}_i)}.
$$

使用 SymPy 计算离散对数，并在本地重现 `print_flag()`：

```python
from sympy.ntheory import discrete_log

P = 0xffffff2f
test_pt = [
    577090037, 2444712010, 3639700191, 3445702192,
    3280387012, 271041745, 1095513148, 506456969,
]
test_ct = [
    3695492958, 1526668524, 3790189762, 20093842,
    2409408810, 239453620, 1615481745, 1887562585,
]
flag_enc = [
    605589777, 4254394693, 463430822, 2146232739,
    4230614750, 1466883317, 31739036, 1703606160,
]

xs = [discrete_log(P, ct, base) for base, ct in zip(test_pt, test_ct)]
print(xs)

plain = b"".join(
    pow(base, int(x), P).to_bytes(4, "little")
    for base, x in zip(flag_enc, xs)
)
print(plain)
```

本地复核得到 8 个指数：

```text
2127877499 1930549411 2028277857 2798570523
901749037 1674216077 3273968005 3294916953
```

第二阶段的 8 个 `uint32_t` 按小端序拼接为：

```text
CrackingDiscreteLogs4TheFun/Lols
```

程序本身打印前缀 `sigpwny{`，并把未以 NUL 结尾的 32 字节数组交给 `%s`，所以运行时可能在右花括号前后继续输出少量栈垃圾。有效内容长度应以 8 个 `uint32_t`，即 32 字节为准。赛事标准提交为：

```text
uiuctf{CrackingDiscreteLogs4TheFun/Lols}
```

## 方法总结

- 核心技巧：识别逐元素模幂验证，将每个输入还原为独立的离散对数，再复现第二组模幂解码。
- 识别信号：验证函数形如 `pow(g, x, p) == h`，且模数固定、各位置互不影响，通常可以直接调用成熟的离散对数实现。
- 复用要点：解码 `uint32_t` 数组时必须确认端序；程序的字符串越界输出是实现瑕疵，不应把随机栈字节误当成 flag 的一部分。
