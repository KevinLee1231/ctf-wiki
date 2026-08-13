# Tung Tung Tung Sahur

## 题目简述

题目先计算明文整数的三次方 $m^3$，并保证它小于 RSA 模数 $N$。随后每乘一次 2 就输出一行 `Tung!`，直到数值不小于 $N$；再每减一次 $N$ 就输出一行 `Sahur!`，直到结果重新落入 $[0,N)$，最后给出 $e=3$、$N$ 和余数 $C$。输出行数完整泄露了这两个循环执行的次数，因此整个变换可直接逆转，不需要分解 $N$。

## 解题过程

设输出中 `Tung!` 的数量为 $t$，`Sahur!` 的数量为 $s$。程序最终给出的数满足：

$$
C = m^3 \cdot 2^t - sN.
$$

题目实例中 $t=164$、$s=1$。先加回减去的模数，再除去所有 2，即可恢复精确的 $m^3$：

$$
m^3 = \frac{C+sN}{2^t}.
$$

下面的脚本直接从 `output.txt` 统计操作次数并解析末尾参数，避免手工复制超长整数：

```python
import re
from pathlib import Path

from Crypto.Util.number import long_to_bytes
from gmpy2 import iroot

lines = Path("output.txt").read_text(encoding="utf-8").splitlines()
tungs = sum(line == "Tung!" for line in lines)
sahurs = sum(line == "Sahur!" for line in lines)

values = {}
for line in lines:
    match = re.fullmatch(r"(e|N|C) = (\d+)", line)
    if match:
        values[match.group(1)] = int(match.group(2))

lifted = values["C"] + sahurs * values["N"]
assert lifted % (1 << tungs) == 0
power = lifted >> tungs

message, exact = iroot(power, values["e"])
assert exact
print(long_to_bytes(int(message)))
```

整数三次根是精确根，转换回字节后得到：

```text
grey{tUn9_t00nG_t0ONg_x7_th3n_s4hUr}
```

## 方法总结

- 核心技巧：把日志行数视为运算轨迹，按相反顺序加回 $N$、除以 2，再取精确整数根。
- 识别信号：当程序在每轮可逆运算时输出固定文本，输出次数本身就是恢复内部状态所需的旁信息。
- 复用要点：不要把最终 $C$ 直接当作标准 RSA 密文；先根据程序真实执行顺序建立等式，并用整除与 `iroot` 的精确标志验证逆推无误。
