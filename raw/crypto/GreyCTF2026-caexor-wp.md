# caexor

## 题目简述

服务要求提交一个偶数长度、至少 $24$ 字符、位于 `a-z{|}` 字母表内的字符串 $s$，使自定义 `caexor(s)` 等于固定的 $16$ 位 base-$29$ 目标 `gimmeflagthankuu`。成功条件还要求输入不能以 `a` 开头，也不能含 `{`、`|`（实现没有额外禁止 `}`）；远端的 `LEN=24`，因此要得到 flag 必须找到长度不超过 $24$ 的原像。

`word` 将 `a` 到 `}` 映射为 $0$ 到 $28$，把 $16$ 个字符视为大端 base-$29$ 整数并在 $B=29^{16}$ 下做加法和乘法。每轮处理两个输入字符：先加常量 $c$，乘常量 $f$，最后仅对最低两个 base-$29$ digit 做逐 digit XOR（结果再对 $29$ 取模）。乘法造成长程耦合，而最后的两字符扰动很小；把扰动视为小整数并求模线性近似的短向量是决定性障碍，故归入 `crypto`。

## 解题过程

### 将迭代改写为关于 XOR 扰动的同余

令初始状态、常量和目标对应的 base-$29$ 整数分别为 $h,c,f,T$。第 $i$ 个输入二元组在乘法后的状态记为 $q_i$，XOR 后的状态记为 $h_{i+1}$，定义：

$$
v_i=h_{i+1}-q_i\pmod{B}.
$$

`word(16, 'a'*14 + pair)` 只影响最后两个 digit，所以每个 $v_i$ 来自很小且可枚举的二字符选择。尽管 digit XOR 不是模加法，定义出的 $v_i$ 使状态递推变为线性：

$$
h_{i+1}=f(h_i+c)+v_i\pmod{B}.
$$

展开 $n$ 轮得到：

$$
\sum_{i=0}^{n-1} f^{n-i-1}v_i
\equiv
T-hf^n-fc\frac{1-f^n}{1-f}\pmod{B}.
$$

远端长度限制给出 $n=12$ 个二元组。这里的未知量不是直接猜 $24$ 个字符，而是寻找能令上述同余残差为零的一组短 $v_i$。

### 构造 CVP 嵌入并执行 BKZ

设右边为 $R$。构造的行格含有每个 $f^{n-i-1}$、$-R$ 与模数 $B$；选择第 $i$ 个基本向量 $v_i$、`-R` 行一次、再加任意 $B$ 行，会产生首坐标

$$
\sum_i f^{n-i-1}v_i-R+kB.
$$

同时在其余坐标保留各个 $v_i$ 和一个锚定的 $1$。列缩放让 BKZ 偏好首坐标为零、其余坐标小的解。官方脚本的核心矩阵如下：

```python
from sage.all import Matrix, identity_matrix, vector

B = 29**16
R = (TARGET - h * f**n - f * c * (1 - f**n) * pow(1 - f, -1, B)) % B
M = Matrix.column([f**(n - i - 1) for i in range(n)] + [-R, B])
M = M.augment(identity_matrix(n + 1).stack(vector([0] * (n + 1))))
Q = Matrix.diagonal([2**128] + [2**4] * n + [2**8])
reduced = (M * Q).BKZ() / Q
```

从规约基中保留首坐标为零且最后坐标绝对值为 $1$ 的行，统一符号后，行的中间 $n$ 个坐标即候选 `v_i`。从 $n=12$ 开始即可得到短解；使用 BKZ 而非 LLL 能更稳定地命中该嵌入中的近向量。

### 将扰动反解回输入字符并验收

格给出的只是加法差值，不能直接把它当作两个字符。对每一轮，先以当前状态计算

$$
q_i=f(h_i+c)\pmod{B},
$$

再枚举 $29^2$ 个低位 digit 对 $(a,b)$，计算实现中的逐 digit结果：

```python
for a in range(29):
    for b in range(29):
        w0, w1 = q_i % 29, (q_i // 29) % 29
        trial = (
            (q_i // 29**2) * 29**2
            + ((w1 ^ b) % 29) * 29
            + ((w0 ^ a) % 29)
        ) % B
        if trial == (q_i + v_i) % B:
            pair = (chr(ord("a") + b), chr(ord("a") + a))
            break
    else:
        continue
    break
else:
    raise ValueError("no valid digit pair")
```

这一步既还原了每对输入字符，也严格检查格候选确实对应真实的 base-$29$ XOR，而非仅满足放松后的同余。得到的 $24$ 字符原像为：

```text
cjpfbkajavhtbncidaanbvab
```

它不以 `a` 开头、不含禁用字符，且重新运行 `caexor` 的输出为 `gimmeflagthankuu`。向服务提交该原像后，长度恰好等于 `LEN`，获得 `grey{why_lattice_enumerate_when_you_can_bkz}`。

## 方法总结

- 核心技巧：把非线性的低位 XOR 单独吸收到小扰动 $v_i$ 中，将多轮乘法递推化为一个模线性 closest-vector 问题，再用小域枚举恢复精确字符。
- 识别信号：固定大进制状态、线性乘加主干、仅少数低 digit 受限非线性扰动以及短原像限制同时出现时，应优先尝试“线性展开 + 格 + 局部枚举”。
- 复用要点：格负责恢复近似/差值，不替代原始运算验证；对低维非线性部分保留一个完整枚举和正向重算，可以消除 BKZ 候选或 digit 端序带来的假解。
