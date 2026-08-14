# bi0sCTF 2022 - bad2code

## 题目简述

题目先用一个 176 位线性同余生成器（LCG）更新状态，再取状态的高 88 位与字符下标、flag 字节异或：

$$
s_{i+1}=(a s_i+c)\bmod 2^{176},\qquad
y_i=(s_{i+1}\mathbin{\gg}88)\oplus i\oplus \operatorname{ord}(f_i),
$$

其中 $a=\texttt{0xBAD2C0DE}$、$c=\texttt{0x6969}$。每个 $y_i$ 又被一层类似 Merkle--Hellman 的背包加密包裹。公开超递增序列为

$$
W=(1,2!,3!,\ldots,90!),\qquad q=\sum W_i,qquad B_i=rW_i\bmod q,
$$

密文文件保存每个整数的有效位长和背包和。最终目标是依次剥掉背包层，并利用已知 flag 前缀 `bi0s` 恢复 LCG 种子。

## 解题过程

### 还原背包明文

题目生成阶段会输出与 $q$ 互素的乘数 $r$，官方解题脚本使用的值为：

```python
r = 439336960671443073145803863477
```

对背包密文 $C$ 计算

$$
C'=C\cdot r^{-1}\bmod q,
$$

就重新得到超递增序列上的子集和。由最大项向最小项贪心相减即可恢复被选中的下标。加密时二进制字符串按最高位到最低位依次乘 $B_0,B_1,\ldots$，所以还要结合密文中保存的位长，将这些下标重新映射成整数：

```python
def decrypt_knapsack(total, bit_length, W, r, q):
    value = total * inverse_mod(r, q) % q
    chosen = []

    for i in range(len(W) - 1, -1, -1):
        if value >= W[i]:
            value -= W[i]
            chosen.append(i)

    assert value == 0
    top = max(chosen)
    out = sum(1 << (top - i) for i in chosen)
    return out << (bit_length - out.bit_length())
```

对 `ct.txt` 中的所有二元组执行这一过程，得到 LCG 层的整数序列 `ciphertext`。

### 用已知前缀恢复 LCG 种子

对于已知的前四个字符 `bi0s`，可以得到四个状态高位：

$$
h_i=\texttt{ciphertext}[i]\oplus i\oplus \operatorname{ord}(\texttt{"bi0s"}[i]).
$$

每个完整状态可写成

$$
s_{i+1}=h_i2^{88}+k_i,\qquad 0\le k_i<2^{88}.
$$

把 LCG 递推展开，就得到关于初始种子 $s_0$ 和四个低位未知量 $k_i$ 的线性模方程组：

$$
a^{i+1}s_0+c\sum_{j=0}^{i}a^j-h_i2^{88}-k_i\equiv0\pmod{2^{176}}.
$$

官方 solver 将这些等式构造成带变量界的格，使用 LLL 后调用 CVP 最近向量恢复 $s_0$。核心建模如下；`solve_linear_mod` 是官方脚本中基于 `fpylll` 的 LLL/CVP 实现：

```python
m = 1 << 176
a = 0xBAD2C0DE
c = 0x6969

s0 = var("s0")
ks = [var(f"k{i}") for i in range(4)]
bounds = {s0: 1 << 176, **{k: 1 << 88 for k in ks}}
equations = []

state_expr = s0
for i, (known, k) in enumerate(zip(b"bi0s", ks)):
    state_expr = a * state_expr + c
    high = ciphertext[i] ^ i ^ known
    equations.append((state_expr == (high << 88) + k, m))

seed = solve_linear_mod(equations, bounds)[s0]
```

最后从恢复出的种子重新运行 LCG，并反向异或：

```python
state = int(seed)
plain = []
for i, value in enumerate(ciphertext):
    state = (state * a + c) % m
    plain.append(chr((state >> 88) ^ i ^ value))

print("".join(plain))
```

输出为：

```text
bi0sctf{lcg_is_good_until_you_break_them_!!}
```

## 方法总结

本题是“超递增背包可逆 + 截断 LCG 状态恢复”的组合。第一层并没有隐藏超递增结构；乘上 $r^{-1}$ 后即可贪心解背包。第二层虽然只泄露每个状态的高 88 位，但已知前缀同时给出连续状态的近似值，未知低位都有明确上界，因此适合转成隐藏数问题，用 LLL/CVP 一次恢复种子。处理这类多层密码题时，应先剥离完全可逆的表示层，再针对真正的信息泄漏建立方程。
