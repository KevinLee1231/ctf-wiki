# Cython-Strike: Bomb Defusion

## 题目简述

Python 扩展 `bomb_defuse` 暴露了 `Bomb` 和 `DefuseKit`。通过 `read_memory` 可逐字节读取炸弹的指令 RAM，但部分地址会失败，得到带缺字的 C 源码。关键陷阱是 C 与 Python 对 `input ^ mask == 114` 的运算符优先级不同。

## 解题过程

先扫描 `0x000` 到 `0x3ff`，对无法读取的位置保留 `*`，即可恢复足够多的校验逻辑：

```python
from bomb_defuse import Bomb, DefuseKit

def dump_instruction_ram(kit):
    result = []
    for address in range(0x400):
        try:
            value = kit.read_memory(address)
            result.append(chr(int(value, 16)) if value != "0x0" else "")
        except Exception:
            result.append("*")
    return "".join(result)

bomb = Bomb()
kit = DefuseKit(bomb)
print(dump_instruction_ram(kit))
print("mask =", bomb.mask)
```

缺字代码的有效部分为：

```c
#define MAX_VAL 0xFFFFFF
unsigned int mask;

void check_pwd(unsigned int input) {
    if (input > MAX_VAL)
        explode_bomb();

    if ((input ^ mask == 114) &&
        (input << ((mask % 5) + 1) == 60578736)) {
        defuse_bomb();
        return;
    }
    explode_bomb();
}
```

在 C 中，相等运算符 `==` 的优先级高于按位异或 `^`，所以第一项实际解析为：

```c
input ^ (mask == 114)
```

本题 `mask = 7355608`，显然不等于 114，于是该项退化成 `input ^ 0`，只要求 `input` 非零。真正决定密码的是第二项：

$$
7355608 \bmod 5 + 1 = 4,
$$

$$
input = 60578736 \mathbin{\text{>>}} 4 = 3786171.
$$

它同时满足 `input <= 0xffffff`。提交即可：

```python
password = 60578736 >> ((bomb.mask % 5) + 1)
flag = bomb.enter_pwd(password)
print(flag)
```

输出：

```text
moectf{CoUnter_TerR0rists_w1n}
```

## 方法总结

从损坏源码恢复逻辑时，不要让语言模型替你“修正”看似奇怪的表达式。`input ^ mask == 114` 在 Python 中会按 `(input ^ mask) == 114` 解释，而 C 中恰好相反；本题密码正是利用这项优先级差异得到的。
