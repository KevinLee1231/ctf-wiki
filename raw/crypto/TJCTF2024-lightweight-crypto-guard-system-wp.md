# lightweight-crypto-guard-system

## 题目简述

题目用线性同余生成器

$$
x_{i+1}=ax_i+c\pmod m
$$

产生状态流。加密每个 flag 字符时先公开当前状态，然后再取后续 $n-1$ 个状态与字符依次异或。下一字符公开的状态恰好位于前一个公开状态之后 $n$ 步。参数 $a,c,m,n$ 未直接给出，其中 $c$ 只有 15 bit。

## 解题过程

把每个字符开头泄漏的状态记为 $y_i=x_{in}$。跨越 $n$ 步后，它本身仍是一个 LCG：

$$
y_{i+1}=A y_i+C\pmod m,
$$

其中 $A=a^n\bmod m$，$C=c(1+a+\cdots+a^{n-1})\bmod m$。因此先对连续的 `leaks` 使用经典 LCG 恢复法：相邻差分 $d_i=y_{i+1}-y_i$ 满足

$$
d_{i+2}d_i-d_{i+1}^2\equiv0\pmod m,
$$

对多个该表达式取最大公因数可恢复 $m$，随后由模逆求得 $A,C$。

接着在模 $m$ 下求 $A$ 的 $n$ 次根，得到原始乘数 $a$ 的候选。因为 $c<2^{15}$，直接穷举 $c$，从第一个泄漏状态推进 $n$ 步，检查是否精确到达第二个泄漏状态。确定参数后按题目状态消费顺序重放即可：

```python
def step(x, a, c, m):
    return (a * x + c) % m

def find_increment(first, second, a, n, m):
    for c in range(1 << 15):
        x = first
        for _ in range(n):
            x = step(x, a, c, m)
        if x == second:
            return c
    raise ValueError("increment not found")

def decrypt(leaks, encrypted, a, c, m, n):
    out = bytearray()
    for leak, value in zip(leaks, encrypted):
        x = leak
        plain = value
        for _ in range(n - 1):
            x = step(x, a, c, m)
            plain ^= x
        out.append(plain)
    return bytes(out)
```

完整明文是一段经过 leetspeak 改写的《星球大战》开场文字。其首尾为：

```text
tjctf{1t_15_a_p3r1od_of_c1v1l_war5_1n_th3_galaxy...c3rta1n_doom_for_th3_champ1on5_of_fr33dom.}
```

仓库中恢复出的完整 flag 为：

```text
tjctf{1t_15_a_p3r1od_of_c1v1l_war5_1n_th3_galaxy._a_brav3_all1anc3_of_und3rground_fr33dom_f1ght3r5_ha5_chall3ng3d_th3_tyranny_and_oppr3551on_of_th3_aw35om3_galact1c_3mp1r3._5tr1k1ng_from_a_fortr355_h1dd3n_among_th3_b1ll1on_5tar5_of_th3_galaxy,_r3b3l_5pac35h1p5_hav3_won_th31r_f1r5t_v1ctory_1n_a_battl3_w1th_th3_pow3rful_1mp3r1al_5tarfl33t._th3_3mp1r3_f3ar5_that_anoth3r_d3f3at_could_br1ng_a_thou5and_mor3_5olar_5y5t3m5_1nto_th3_r3b3ll1on,_and_1mp3r1al_control_ov3r_th3_galaxy_would_b3_lo5t_for3v3r._to_cru5h_th3_r3b3ll1on_onc3_and_for_all,_th3_3mp1r3_15_con5truct1ng_a_51n15t3r_n3w_battl3_5tat1on._pow3rful_3nough_to_d35troy_an_3nt1r3_plan3t,_1t5_compl3t1on_5p3ll5_c3rta1n_doom_for_th3_champ1on5_of_fr33dom.}
```

## 方法总结

- 稀疏泄漏的状态仍可视为“抽样后的 LCG”，先恢复步长为 $n$ 的等价参数，再反推原参数。
- 小范围增量 $c$ 是最后的验证锚点；只靠模 $n$ 次根可能产生多个 $a$ 候选，必须用相邻泄漏重放确认。
- 解密时最容易出现 off-by-one：泄漏值是本字符的第一个状态，只应再推进并异或 $n-1$ 次。
