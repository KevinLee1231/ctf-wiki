# Library of Babel

## 题目简述

服务把每一页表示为 3200 个字符，字母表共有 39 个符号：空格、句点、逗号、数字和小写字母。用户输入四个大整数坐标 $(w,x,y,z)$，以及 side、shelf、book、page；程序把这些值组合为线性同余生成器（LCG）的初始状态，快速前进 $2^{512}$ 步，再把最终状态按自定义 base39 映射显示为页面。

只有页面恰好为：

```text
this page cannot be found.<其余位置全部为空格>
```

时才会输出 flag。题目同时含有 LCG 数学，但决定性工作是从二进制恢复字符表、坐标拼接顺序和各输入字段如何生成参数，因此归入 Reverse。

## 解题过程

### 恢复状态与坐标编码

令：

$$
B=39^{800},\qquad m=39^{3200}=B^4.
$$

程序先把每个坐标模 $B$，再按四个 base-$B$“数位”拼接：

$$
x_0=((wB+x)B+y)B+z.
$$

LCG 参数由其余四项决定：

$$
a=39\cdot(\text{shelf}\cdot\text{side})+1,
$$

$$
c=32\cdot\text{page}+\text{book},
$$

并迭代 $N=2^{512}$ 次：

$$
x_{i+1}=ax_i+c\pmod m.
$$

为简化参数，取 side、shelf、book、page 全为 1，于是 $a=40$、$c=33$。因为 $\gcd(40,39^{3200})=1$，乘数在模 $m$ 下可逆。

### 把目标页面映射为整数

自定义字符到数位的映射为：空格 $\mapsto0$、`.` $\mapsto1$、`,` $\mapsto2$、`0` 至 `9` $\mapsto3$ 至 $12$、`a` 至 `z` $\mapsto13$ 至 $38$。按高位在前累积即可：

```python
def page_to_int(text):
    value = 0
    for ch in text:
        if ch == " ":
            digit = 0
        elif ch == ".":
            digit = 1
        elif ch == ",":
            digit = 2
        elif "0" <= ch <= "9":
            digit = ord(ch) - ord("0") + 3
        else:
            digit = ord(ch) - ord("a") + 13
        value = value * 39 + digit
    return value

target_page = "this page cannot be found.".ljust(3200, " ")
x_final = page_to_int(target_page)
```

### 快速倒退 LCG

单步逆变换是：

$$
x_i=a^{-1}x_{i+1}-a^{-1}c\pmod m.
$$

因此，把新乘数设为 $a^{-1}\bmod m$，新增量设为 $-a^{-1}c$，就能复用快速跳步公式倒退 $N$ 步。为避免在模环中直接除以 $a-1$，计算几何级数时先在模 $(a-1)m$ 下求幂：

```python
def skip(a, c, m, x, steps):
    if steps < 0:
        a_inv = pow(a, -1, m)
        a, c, steps = a_inv, -a_inv * c, -steps

    a_minus_1 = a - 1
    geom = (pow(a, steps, a_minus_1 * m) - 1) // a_minus_1
    return (pow(a, steps, m) * x + geom * c) % m

m = 39 ** 3200
x_initial = skip(40, 33, m, x_final, -(2 ** 512))
```

最后按拼接的相反顺序每次对 $B$ 取模：

```python
B = 39 ** 800
z = x_initial % B
x_initial //= B
y = x_initial % B
x_initial //= B
x = x_initial % B
x_initial //= B
w = x_initial % B

print(w, x, y, z, sep="\n")
```

向服务依次提交 $w,x,y,z$，再把 side、shelf、book、page 都设为 1。服务正向跳过 $2^{512}$ 步后会得到目标页面，并返回：

```text
uiuctf{th3_l18br4ry_1s_unlim1t3d_bu7_p3r10d1c_c9176412}
```

## 方法总结

- 核心技巧：把四维坐标还原为模 $39^{3200}$ 的单一 LCG 状态，再利用乘数可逆性和快速跳步公式从目标页面倒推初始坐标。
- 识别信号：程序处理超大十进制坐标、GMP 大整数、固定指数跳步和 base39 页面；页面查找本质上是可逆状态映射，而不是穷举无限图书馆。
- 复用要点：先核对字符表顺序、左/右补位和坐标端序；倒推前必须验证 $\gcd(a,m)=1$。几何级数中的整数除法不能随意改成模逆，应使用题目实现所采用的扩展模数技巧，或用仿射变换快速幂组合。
