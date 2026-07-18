# f, l and ag++

## 题目简述

服务生成 2048-bit RSA 模数 `N`，公钥指数 `e=257`，再把一段 68 字节随机值切成 17、17、34 字节的 `f`、`l`、`ag`。选手得到四个 textbook RSA 密文：

```text
f_enc    = f^e mod N
l_enc    = l^e mod N
ag_enc   = ag^e mod N
flag_enc = (f || l || ag)^e mod N
```

数学结构与 Dreamhack Invitational 2024 的 `f, l and ag` 相同，但本题只给 10 秒。直接求高次二元 resultant 或 Gröbner basis 来不及，需把 resultant 在许多点并行求值，再用插值重建消元多项式。

## 解题过程

### 归一化并只保留两个比值未知量

68 字节拼接整数满足：

```text
flag = 256^51*f + 256^34*l + ag
```

在 `Z/NZ` 中除以 `f`，定义：

```text
r = 256^17 + l/f
s = ag/f
```

于是：

```text
flag/f = s + 256^34*r
```

从四个密文可直接计算：

```python
F.<x> = Zmod(N, is_field=True)[]
l2, ag2, flag2 = [F(v) / f_enc for v in (l_enc, ag_enc, flag_enc)]
```

并得到三条一元关系：

```text
(r - 256^17)^e = l2
s^e             = ag2
(s + 256^34*r)^e = flag2
```

这里 `is_field=True` 是让 Sage 对 `Zmod(N)` 多项式执行域式欧几里得运算。随机系数几乎总与 `N` 互素；若真的遇到不可逆系数，反而会暴露 `N` 的因子。

### 把昂贵 resultant 变成并行点值

固定一个候选 `r=i`，以 `s` 为多项式变量构造：

```text
B(s) = s^e - ag2
A(s) = (s + 256^34*i)^e - flag2 - B(s)
```

减去 `B` 后，`A` 的最高次项被消掉。真实 `r` 下二者有公共根 `s`，所以：

```text
R(r) = Res_s(A, B) = 0
```

对单个 `i`，用多项式 Euclidean/subresultant remainder sequence 只求一个标量 resultant：

```python
@parallel(ncpus=20)
def get_resultant(i):
    B = x^e - ag2
    A = (x + 256^34*i)^e - flag2 - B
    scale = 1
    while A.degree():
        scale *= B.lc()
        A, B = B, A % B
    value = scale^2 * A[0]^B.degree()
    return F(i)^e, value
```

该 resultant 只依赖 `r^e`，把 `y=r^e` 当插值变量后次数至多为 `e`。因此并行计算 `i=0..e` 的 `e+1=258` 个点，就足以恢复 `R`：

```python
arr = [item for _, item in get_resultant([0..e])]

# 在点 y=i^e 上重建 R_y(y)
R_y = vector(CRT_basis([x-y for y, _ in arr])) \
    * vector(value for _, value in arr)

# 把 y 替换成 x^e：系数之间插入 e-1 个 0
R_x = F([coef for c in R_y for coef in [c] + [0]*(e-1)])
```

这种“点值 resultant + Lagrange/CRT 插值”避免在符号参数 `r` 上直接维护巨大的高次系数，是满足 10 秒限制的核心优化。PDF 记录的实现使用 20 个 CPU 进程，约 5 秒完成；具体耗时仍取决于核心数和 Sage 版本。

### 两次 gcd 与有理重构

真实 `r` 同时满足 `R_x(r)=0` 和 `(r-256^17)^e-l2=0`。两者的 gcd 应为线性因子 `x-r`：

```python
r = -gcd(R_x, (x - 256^17)^e - l2)[0]
```

再代回另外两条关系，恢复 `s`：

```python
s = -gcd(x^e - ag2, (x + 256^34*r)^e - flag2)[0]
```

此时：

```text
r - 256^17 = l/f mod N
```

而 `l`、`f` 都只有 136 bit，远小于约 2048-bit 的 `N`，可以用有理重构唯一找回小分子、小分母：

```python
l, f = rational_reconstruction(r - 256^17, N).as_integer_ratio()
ag = ZZ(f * s)
```

按固定宽度拼回原始 68 字节值，不能用最短长度编码，否则前导零会改变拼接：

```python
challenge_flag  = int(f).to_bytes(17, 'big')
challenge_flag += int(l).to_bytes(17, 'big')
challenge_flag += int(ag).to_bytes(34, 'big')
```

服务端的输入协议还有一层易错点：它先执行 `bytes.fromhex(input).decode()`，再与 `hex(flag_int)[2:]` 比较。因此要发送“68 字节值对应整数的十六进制文本”本身的十六进制编码：

```python
answer_text = hex(bytes_to_long(challenge_flag))[2:].encode()
io.sendline(answer_text.hex().encode())
```

校验通过后得到：

```text
RCTF{hah_i_only_wanted_to_steal_your_high_degree_speedrun_techniques__:P}
```

原题作者的 [f, l and ag 说明](https://soon.haari.me/flandag/) 展示了较慢的二元 Franklin–Reiter 消元路线：把一个变量视作系数环中的常量，用伪余式逐次降低另一变量次数，并不断模已知多项式压缩系数。本题正文给出的点值 resultant 与插值，是同一消元目标在严格时间限制下的高速实现。

## 方法总结

固定分段长度把拼接关系变成了 related-message 方程；textbook RSA 又保留了这些代数关系。常规消元能证明可解，却未必满足比赛时限。遇到“resultant 数学上正确但符号计算过慢”时，可以检查结果对参数是否只依赖某个低次组合，在多个标量点并行求值后再插值。最后的有理重构利用的是分段整数远小于 `N`，而固定字节宽度和题目双重十六进制协议同样属于完整解法的一部分。
