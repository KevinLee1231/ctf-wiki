# bi0sCTF 2022 Eerie Jit Writeup

## 题目简述

Eerie Jit 是一道 64 位 ELF 逆向题。程序先检查 `bi0sCTF{...}` 的外层格式，再把内部 16 字节拆成四个大端 32 位整数。真正的校验指令并不完整存在于静态代码中，而是运行时由一段 JIT 逻辑写入可执行内存；生成出的代码最终只是在有限域上检查四条代数方程。

因此，本题的决定性步骤是从 JIT 生成器中恢复“它会执行什么”，而不是逐条模拟所有机器码。把运行时常量和运算顺序整理成方程后，就能直接求出四个明文块。

## 解题过程

### 恢复输入布局与模数

程序去掉固定前缀后，将剩余 16 字节按大端序解释为 $A$、$B$、$C$、$D$。JIT 代码中的所有运算都对素数

$$
p=\mathtt{0x7EFF4B91}
$$

取模，四个比较常量依次为：

```text
0x1EF6E9EB
0x34CC1889
0x68E54823
0x11226D6A
```

仓库里的本地求解草稿省略了这五个初始化常量；它们必须从生成器或官方分析中的 JIT 指令流补回，否则脚本无法独立运行。

### 将 JIT 指令化成方程

顺着生成的乘、加、减和取模操作，可以整理出：

$$
\begin{aligned}
5AB-4A^2+105A+6B &\equiv \mathtt{0x1EF6E9EB}\pmod p,\\
2A^2+13B+17A &\equiv \mathtt{0x34CC1889}\pmod p,\\
5B^2+105C-5BC &\equiv \mathtt{0x68E54823}\pmod p,\\
5C^2-4CD+303D &\equiv \mathtt{0x11226D6A}\pmod p.
\end{aligned}
$$

第二式可以先把 $B$ 表示为 $A$ 的多项式，再代回第一式。得到候选 $A$ 后，第三、第四式分别对 $C$、$D$ 都是一次方程，所以不需要暴力枚举 $2^{128}$ 的输入空间。

### 在 SageMath 中求解

下面的脚本补全常量，并保持所有除法都在 $\mathrm{GF}(p)$ 中进行：

```python
p = 0x7EFF4B91
k1 = 0x1EF6E9EB
k2 = 0x34CC1889
k3 = 0x68E54823
k4 = 0x11226D6A

F = GF(p)
R.<A> = PolynomialRing(F)

B_expr = (F(k2) - 2*A^2 - 17*A) / F(13)
f = 5*A*B_expr - 4*A^2 + 105*A + 6*B_expr - F(k1)

for a in f.roots(multiplicities=False):
    b = (F(k2) - 2*a^2 - 17*a) / F(13)

    den_c = F(105) - 5*b
    if den_c == 0:
        continue

    c = (F(k3) - 5*b^2) / den_c
    den_d = F(303) - 4*c
    if den_d == 0:
        continue
    d = (F(k4) - 5*c^2) / den_d

    chunks = [int(x).to_bytes(4, "big") for x in (a, b, c, d)]
    inner = b"".join(chunks)
    if inner.endswith(b"}") and all(0x20 <= x < 0x7f for x in inner):
        print(b"bi0sCTF{" + inner)
```

脚本先检查两次除法的分母，再以可打印字符和结尾花括号筛选有限域中的候选根。整理后的候选块为：

```text
A = 0x74696d65  -> time
B = 0x6c617073  -> laps
C = 0x696e675f  -> ing_
D = 0x6a69747d  -> jit}
```

代回四式均与比较常量一致，最终 flag 为：

```text
bi0sCTF{timelapsing_jit}
```

官方赛后文章展示了从 JIT dump 识别算术模板以及用 Sage 求解的原始过程：[Eerie Jit 官方题解](https://blog.bi0s.in/2023/01/25/RE/Eerie_Jit-bi0sCTF22/)。

## 方法总结

JIT 逆向的重点是区分“生成器的控制逻辑”和“生成代码的业务逻辑”。先标记写入可执行缓冲区的指令模板，再把重复的寄存器搬运、乘加和取模压缩成代数表达式，题目就从动态机器码分析降维成了有限域求根。

本题还暴露了一个常见归档风险：官方或仓库中的求解草稿未必自包含。只有补上模数与四个比较常量、明确大端分块，并把候选重新代回原方程，WP 才具备可复现性。
