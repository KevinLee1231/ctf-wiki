# ecb_oracle

## 题目简述

服务端把用户字符串 `x` 按一个由字符串内容决定的位置切开，将 flag 插入中间，再用会话内固定随机密钥执行 AES-ECB 和 PKCS#7 填充：

```python
def transform_input(x, secret=FLAG):
    separator = generate_separator(x)
    return x[:separator] + secret + x[separator:]
```

ECB 会把相同的 16 字节明文块映射为相同密文块，因此可以逐字节比较候选块。额外障碍是插入位置并非直接可控，而是由输入各字符的线性项和平方项计算得出；必须先反解一个能产生指定 `separator` 的字符串。

## 解题过程

### 还原插入位置函数

令输入长度为 $n$，字符整数值为 $x_i$，并令 $M=n+1$，则服务端计算

$$
L=\sum_{i=0}^{n-1}(i+1)x_i,\qquad
S=\sum_{i=0}^{n-1}x_i^2,
$$

$$
\operatorname{separator}(x)=(L+S+n)\bmod(n+1).
$$

为了构造 `separator(x) = k`，官方解法令 $n=k$，将前 $n-1$ 个字符都固定为同一个大写字母 $\alpha$，只枚举最后一个字符 $y$。此时

$$
L_{\text{prefix}}=\alpha\frac{(n-1)n}{2},\qquad
S_{\text{prefix}}=(n-1)\alpha^2.
$$

因为目标余数就是 $n$，只需令

$$
L_{\text{prefix}}+S_{\text{prefix}}+ny+y^2\equiv0\pmod{n+1}.
$$

模数只有 $n+1$，可以枚举 $\alpha$ 和 $y\bmod(n+1)$，再选取同余且可打印的 ASCII 字符：

```python
import string

def generate_separator(x):
    n = len(x)
    modulus = n + 1 if n else 1
    linear = sum(ord(ch) * (i + 1) for i, ch in enumerate(x))
    square = sum(ord(ch) ** 2 for ch in x)
    return (linear + square + n) % modulus

def string_for_separator(separator):
    if separator == 0:
        return ""

    n = separator
    modulus = n + 1
    for fixed in string.ascii_uppercase:
        alpha = ord(fixed)
        prefix_term = alpha * ((n - 1) * n // 2)
        prefix_term += (n - 1) * alpha * alpha

        for residue in range(modulus):
            if (residue * residue + n * residue + prefix_term) % modulus:
                continue
            for value in range(33, 127):
                if value % modulus == residue:
                    candidate = fixed * (n - 1) + chr(value)
                    assert generate_separator(candidate) == separator
                    return candidate
    raise RuntimeError("no printable preimage")
```

### 逐字节恢复 flag

已知 flag 前缀为 `shellmates{`。设当前已恢复长度为 $i$，先计算让下一个未知字节落在 AES 块末尾所需的插入位置：通常取

$$
k=15-(i\bmod16).
$$

官方 solver 在 $k=0$ 的边界处改用后续块的等价插入位置，避免空输入被服务拒绝。用 `string_for_separator(k)` 生成参考输入后，服务端明文前部为“构造前缀 + 真 flag”，可以取得包含下一个未知字节的参考密文块。

对每个可打印候选字符 $c$，再构造以“前缀 + 已恢复 flag + $c$”开头的测试输入，并追加填充字符，直到服务端计算出的插入位置位于整段猜测之后。这样测试密文的对应块完全由可控候选组成。若该块与参考块相等，ECB 的确定性说明候选字节正确。

核心循环可概括为：

```python
known = "shellmates{"

while not known.endswith("}"):
    crafted, block_index = make_reference_input(known)
    reference = oracle(crafted)[block_index]

    for ch in PRINTABLE_CHARS:
        test = make_safe_test_input(crafted, known + ch)
        if oracle(test)[block_index] == reference:
            known += ch
            break
    else:
        raise RuntimeError("no candidate matched")
```

逐字节比较最终恢复出：

```text
shellmates{EcB_w1ll_alW4Y$_b3_THE_w34KEsT_aEs_MOd3}
```

本题官方 PDF 共 6 页。逐页对照可确认其中的题目信息、`separator` 数学反演、构造算法、Python 实现、ECB 逐字节流程和最终 flag 都已转写；PDF 仅包含文字、公式和代码，没有需要作为独立视觉证据保留的图片。

## 方法总结

- 核心技巧：先求输入变换的可控原像，再执行 AES-ECB byte-at-a-time 块匹配。
- 识别信号：只要同一密钥下可反复查询 ECB，并能让未知秘密与可控字节进入同一固定块，就应考虑逐字节字典攻击；秘密不在末尾也不必然阻止攻击。
- 复用要点：必须同时核对插入位置、目标块编号和 16 字节边界；测试候选时还要保证真实 flag 被插入到候选段之后，否则比较的并不是同一明文块。
