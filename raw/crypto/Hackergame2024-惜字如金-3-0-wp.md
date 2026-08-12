# 惜字如金 3.0

## 题目简述

题目给出三份被“惜字如金化”的 Python 源码：单词末尾位于辅音之后的 `e/E` 会被删除，连续重复的同一辅音（忽略大小写）只保留第一个。原文件每行恰好补齐到 80 字节，服务端逐行比较上传内容；错误响应会泄露哈希不匹配、长度不匹配或最后一个不匹配字节。

三份程序结构相同。每行先经 48 位 CRC，再经过二次多项式

$$
H(t)=u_2t^2+u_1t+u_0\pmod {2^{48}},
$$

最后把 6 字节小端结果编码为 Base85 文件名。A 是语法补全；B 要从哈希提示反解 CRC 多项式；C 则要构造指定 CRC 余数，借文件名碰撞读出 `answer_c.txt`。决定性主障碍是 CRC、模 $2^{48}$ 二次同余和 $\mathrm{GF}(2)$ 多项式运算，因此归为 `crypto`。

## 解题过程

### 题目 A：按 Python 语义补全

短化规则是确定的，但逆变换不唯一。这里利用 Python 语法、类型名和 API 名称恢复被删字符，例如：

```python
import atexit, base64, flask, itertools, os, re

assert len(poly) == poly_degree + 1
for _ in range(8):
    digest = (digest >> 1) ^ (flip if digest & 1 else 0)
return digest.to_bytes(6, "little")

with open(__file__, "rb") as f:
    paths.append(path)
    pf.write(row)
```

其余 `Response`、`content_type`、`enumerate`、`None`、`remove`、`FileNotFoundError`、`pass`、`b85encode`、`b85decode`、`decode` 等同理恢复。提交前必须逐行补空格到 80 字节；否则语义正确也会触发 `Unmatched length`。

### 题目 B：由二次同余恢复 CRC 多项式

选择输入

```python
probe = b"\xff" * 11 + b"\x7f"
```

经 CRC 的初始/末尾异或和小端位序处理后，待除多项式恰为 $1+x+\cdots+x^{48}$。因此余数直接泄露生成多项式的系数补集。服务端返回的 6 字节哈希提示先按小端解释为 $u$，然后解

$$
u_2t^2+u_1t+u_0\equiv u\pmod {2^{48}}.
$$

配方得到

$$
(2u_2t+u_1)^2\equiv u_1^2-4u_2(u_0-u)\pmod {2^{48}}.
$$

用 `sqrt_mod(rhs, 1 << 48, all_roots=True)` 求平方根；对每个根 $s$ 计算

```python
double_t = ((s - u1) * pow(u2, -1, 1 << 48)) % (1 << 48)
candidates = [double_t // 2, double_t // 2 + (1 << 47)]
t = [x for x in candidates if (u2*x*x + u1*x + u0) % (1 << 48) == u]
```

平方根与“除以 2”都会产生增根，必须代回原式筛选。两条候选多项式逐一上传后，正确的源码行为：

```python
poly, poly_degree = "BBbbbBbbBBbbBbBbbbbbBbBBBbBBbbbBBBbBBbbBBbbBBBbBB", 48
```

### 题目 C：构造 CRC 文件名碰撞

C 中大小写被删去的十六进制常量数值其实固定为

```python
u2, u1, u0 = 0xDFFFFFFFFFFF, 0xFFFFFFFFFFFF, 0xFFFFFFFFFFFF
```

直接猜原大小写有 $2^{32}$ 种，不可行。关键是程序会以 `base85(hash(row)) + '.txt'` 创建每行文件，而真正 flag 文件名为 `answer_c.txt`。`b85decode(b"answer_c")` 得到目标 6 字节哈希；由已恢复的二次映射和 CRC 定义，可算出目标 CRC 余数 $r(x)$。官方数据可紧凑记录为指数集合：

$$
r(x)=\sum_{i\in R}x^i,
\quad R=\{1,5,14,16,17,20,24,25,26,27,30,31,32,33,36,37,38,39,40,42,44,46\}.
$$

CRC 生成多项式为 $p(x)=\sum_{i\in P}x^i$，其中

$$
P=\{0,1,2,3,4,7,8,9,10,11,12,13,15,16,17,19,22,27,28,31,32,34,35,36,37,41,43,44,48\}.
$$

希望只修改长度为 $n$ 的请求前 $m$ 字节，使其 CRC 余数变成 $r(x)$。保留其余字节得到低次部分 $n_0(x)$，令

$$
q(x)=x^{8n-8m+48}.
$$

因为 $\gcd(p,q)=1$，扩展欧几里得可求

$$
s(x)p(x)+t(x)q(x)=1.
$$

于是令

$$
m(x)=((r(x)-n_0(x))t(x))\bmod p(x),
$$

再构造

$$
n(x)=(u(x)p(x)+m(x))q(x)+n_0(x),
$$

就保证 $n(x)\bmod p(x)=r(x)$，同时未修改受控前缀之外的字节。`sympy.polys.galoistools` 中的 `gf_gcdex`、`gf_mul`、`gf_add`、`gf_sub` 和 `gf_rem` 可直接实现上述等式。取 $m=7$，枚举 8 次多项式 $u(x)$ 的 256 种值，便能避开会被 HTTP 按行切断的 `\r`、`\n`。

先用该碰撞请求枚举响应中的 `Unmatched length (N)` 确定 `answer_c.txt` 长度；再从末尾向前调整请求字节，通过 `Unmatched data (0x??)` 逐字节恢复内容。前六个 Base85 字节可由已知 flag 前缀 `flag{` 固定，第七字节只需枚举 Base85 字母表的 85 种可能，选择解码后全为可打印 ASCII 且格式闭合的结果。

官方的两张图片都只是源码文本截图：A 图中的补全内容已经转写为上面的 Python 片段，C 图中的十六进制数值也已转写，因此不保留图片。

## 方法总结

- 核心技巧：利用错误提示建立哈希 oracle；先解模 $2^{48}$ 二次同余恢复 CRC 参数，再用 $\mathrm{GF}(2)$ 扩展欧几里得构造指定余数碰撞。
- 识别信号：哈希文件名由可控输入生成、错误分支区分长度/字节、CRC 参数只有 48 位且存在已知目标文件名时，应考虑 chosen-remainder CRC 构造。
- 复用要点：CRC 的端序、初始异或、末尾异或和多项式系数顺序必须统一；模 $2^k$ 开平方及除以 2 会引入多解，候选一定要代回原式；逐行校验还要求精确保留 80 字节填充。
