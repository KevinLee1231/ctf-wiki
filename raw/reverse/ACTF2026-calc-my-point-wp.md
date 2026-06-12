# calc my point

## 题目简述

题目是 Rust + GMP 写的 EML 表达式执行器，核心来自用单个二元算子 `exp(x) - log(y)` 构造初等函数的思想，最终约束落到 CRT 和数值恢复。

## 解题过程

附件解压后得到一个 64 位 ELF 可执行文件：

```bash
unzip calc_my_point.zip
file calc_my_point
chmod +x calc_my_point
```

`file` 输出显示它是 x86-64 PIE ELF，动态链接，未 strip。程序没有去符号，核心函数为 `calc_my_point::main`。在 `main` 中可以看到输入先被读取为无符号整数，并检查范围：

```asm
cmp eax, 0x29b92701
jae fail
```

`0x29b92701 = 700000001`，所以待求学号满足 `0 <= x < 700000001`。

继续分析核心计算逻辑，程序使用了 mpc/mpfr 高精度复数库。通过 `readelf -r` 和符号表可以对应到 `mpc_init3`、`mpc_exp`、`mpc_log`、`mpc_sub`、`mpfr_less_p` 等函数。

程序从 `.rodata` 中读取一张静态表，表偏移为 `0xa200`，长度为 `0x11168` 字节，按 i64 解析后一共有 8749 项，内容只包含 `1`、`0`、`-1` 三种值：

```text
1   出现 4366 次
0   出现 9 次
-1  出现 4374 次
```

这张表是一个 RPN（逆波兰）表达式。把输入记为 `x`，栈机逻辑如下：
stack = []
for v in table:
    if v == 1:
        stack.append(1)
    elif v == 0:
        stack.append(x)
    elif v == -1:
        b = stack.pop()
        a = stack.pop()
        stack.append(exp(a) - log(b))
即 `-1` 对应一个二元运算 $T(a,b)=\exp(a)-\log(b)$。静态表执行结束后栈中只剩一个复数结果 $z$。最后程序构造阈值 $\epsilon=2^{-144}$，然后判断：

```text
abs(real(z) - 100) < eps
abs(imag(z)) < eps
```

满足时输出 `Success`。将 $T(a,b)$ 代回表达式进行化简。关键恒等式：

```text
T(a, 1) = exp(a) - log(1) = exp(a)
T(a, T(b, 1)) = exp(a) - log(exp(b)) = exp(a) - b
```

静态表中绝大部分节点都是常量子树，真正与输入 `x` 有关的节点只有 284 个。将常量子树先折叠，再不断用 `exp/log` 抵消关系化简，最终表达式可以化为：

```text
F(x) = exp(-x) * tanh(x/20 + 3) + 50 * (sin(π*x/32768) + cos(π*x/33033))
```

需要 $F(x)\approx100$。由于当 `x` 较大时 `exp(-x) * tanh(x/20 + 3)` 远小于 $2^{-144}$，不影响最终比较，所以主要条件为：

```text
50 * (sin(π*x/32768) + cos(π*x/33033)) = 100
```

因为 `sin` 和 `cos` 的最大值都是 1，所以必须同时满足：

```text
sin(π*x/32768) = 1  →  x ≡ 16384 (mod 65536)
cos(π*x/33033) = 1  →  x ≡ 0     (mod 66066)
```

使用中国剩余定理（CRT）求解：

```text
令 x = 66066 * k ，代入第一个同余：
66066k ≡ 16384 (mod 65536)
530k   ≡ 16384 (mod 65536)
265k   ≡ 8192  (mod 32768)
求得 k ≡ 8192 (mod 32768) ，因此：
x = 66066 * 8192 = 541212672
```

该解满足范围限制 `541212672 < 700000001`。整体周期为 `lcm(65536, 66066) = 2164850688 > 700000001`，因此范围内只有这一个解。

求解脚本：

```python
#!/usr/bin/env python3
import struct
import sys
from collections import Counter
from math import gcd
BIN = sys.argv[1] if len(sys.argv) > 1 else './calc_my_point'
TABLE_OFF = 0xA200
TABLE_SIZE = 0x11168
LIMIT = 700000001
def crt_pair(a1, m1, a2, m2):
    g = gcd(m1, m2)
    assert (a2 - a1) % g == 0
    m1g, m2g = m1 // g, m2 // g
    t = ((a2 - a1) // g * pow(m1g, -1, m2g)) % m2g
    return (a1 + m1 * t) % (m1 * m2g), m1 * m2g
with open(BIN, 'rb') as f:
    f.seek(TABLE_OFF)
    data = f.read(TABLE_SIZE)
arr = list(struct.unpack('<%dq' % (TABLE_SIZE // 8), data))
### RPN 表校验
assert set(arr) == {-1, 0, 1}
c = Counter(arr)
assert c[0] == 9
assert c[-1] == 4374
assert c[1] == 4366
### 化简后：
### F(x) = exp(-x)*tanh(x/20+3) + 50*(sin(pi*x/32768) + cos(pi*x/33033))
### 需要 F(x) = 100
### sin(pi*x/32768) = 1  ->  x ≡ 16384 (mod 65536)
### cos(pi*x/33033) = 1  ->  x ≡ 0     (mod 66066)
x, period = crt_pair(16384, 65536, 0, 66066)
assert 0 <= x < LIMIT
assert x % 65536 == 16384
assert x % 66066 == 0
assert period > LIMIT
print('entries =', len(arr))
print('counts =', dict(c))
print('x =', x)
print('flag = ACTF{%d}' % x)
```

脚本输出 `entries`、`counts` 和 `x` 后得到 `flag = ACTF{...}`，验证了计数恢复逻辑。

最终得到：

```text
ACTF{541212672}
```

## 方法总结

- 核心技巧：把 EML 运算解释为可建模的数学约束，结合 CRT 和周期性恢复目标整数。
- 识别信号：题目给出奇怪的表达式执行器、`exp/log` 单算子和大整数库时，应优先把程序语义转成数学等式。
- 复用要点：`exp(x) - log(y)` 这类二元算子可以组合出大量初等函数；先确认表达式语言的数值域和精度，再用同余/周期约束收敛搜索范围。
