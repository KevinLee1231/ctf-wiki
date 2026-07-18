# week4奶茶

## 题目简述

附件 `hongbao.exe` 是 32 位 PE，验证过程分为两段。第一段把 10 字节输入映射为一组单调不减的整数，并检查它们的乘积；通过后，程序调用 `VirtualProtect` 修改代码页权限，以 `0x47` 解密一段自修改代码（SMC），再验证 19 字节的第二段输入。

## 解题过程

第一段要求输入长度恰好为 10。程序使用的 10 字节常量为：

```text
03 4d 50 5a 50 24 71 77 7f 7f
```

设输入字符为 $k_i$、对应常量为 $c_i$，程序计算：

$$
v_i=(c_i\oplus\operatorname{ord}(k_i))-0x30
$$

随后要求 $v_i$ 单调不减，并检查 10 个数的乘积等于：

```text
0x00000017592b1b49 = 100280245065
```

分解该整数可得十个互不相同且已经递增排列的质因数：

$$
100280245065=3\times5\times7\times11\times13\times17\times19\times23\times29\times31
$$

因此 $v_i$ 依次取上述十个质数，再由 $k_i=c_i\oplus(v_i+0x30)$ 反推输入，得到第一段：

```text
0xgame2020
```

第一段验证通过后，程序把 `[0x401120, 0x401200)` 的代码逐字节异或 `0x47`。调试时可在解密循环结束后暂停并重新分析该区域。解密后的函数要求第二段长度为 19，并使用下列规则生成比较值：

$$
t_i=
\begin{cases}
(x_i-i)\oplus0x45,&i\text{ 为偶数}\\
(x_i+i)\oplus0x23,&i\text{ 为奇数}
\end{cases}
$$

目标数组位于 `0x4031c4`：

```text
36 4d 74 41 68 5b 1c 59 22 4b 1f 57 1f 50 1e 51 20 5e 65
```

下面的脚本同时恢复两段输入，并复算约束：

```python
from math import prod

constants = bytes.fromhex("03 4d 50 5a 50 24 71 77 7f 7f")
factors = [3, 5, 7, 11, 13, 17, 19, 23, 29, 31]
product_target = 0x17592B1B49

assert factors == sorted(factors)
assert prod(factors) == product_target
part1 = bytes(c ^ (v + 0x30) for c, v in zip(constants, factors))

target = bytes.fromhex(
    "36 4d 74 41 68 5b 1c 59 22 4b 1f 57 1f 50 1e 51 20 5e 65"
)
part2 = bytearray()
for i, value in enumerate(target):
    if i % 2 == 0:
        part2.append((value ^ 0x45) + i)
    else:
        part2.append((value ^ 0x23) - i)

assert bytes(
    ((ch - i) ^ 0x45) if i % 2 == 0 else ((ch + i) ^ 0x23)
    for i, ch in enumerate(part2)
) == target

print(part1.decode())
print(part2.decode())
print(f"0xgame{{{part1.decode()}-{part2.decode()}}}")
```

输出为：

```text
0xgame2020
sm3_1s_so_difficul2
0xgame{0xgame2020-sm3_1s_so_difficul2}
```

## 方法总结

第一阶段不是泛泛地“做因数分解”，而是利用乘积、数量和单调性唯一确定十个 $v_i$，再逐字节逆异或；第二阶段则要在 SMC 解密完成后恢复真实控制流，并按索引奇偶分别逆运算。两段约束都可以用短脚本复算，避免只凭调试器中的中间结果猜测 flag。
