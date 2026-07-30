# L3akCTF 2024 coolVM Writeup

## 题目简述

程序实现了一台 8 指令栈式虚拟机：

```text
psh, mul, div, jmp, cmp, add, sub, pop
```

每个操作码不是单独的枚举值，而是一个位掩码。解释器会遍历 8 个位，只要某位被置位就调用对应的虚函数。因此一字节可以合并多条指令。发布版程序却在同一字节匹配到第二条指令时立即退出，而附件 `code` 正是按“合并指令”生成的；发布版还只允许自定义输入 9 字节，无法直接载入整份代码。

决定性任务是恢复位到指令的映射，并复现被发布版检查阻止的多指令语义。

## 解题过程

源码中 `DEBUG` 版本的指令顺序就是 `code` 使用的映射：

| 位 | 指令 |
|---:|---|
| 0 | `div` |
| 1 | `psh` |
| 2 | `mul` |
| 3 | `jmp` |
| 4 | `pop` |
| 5 | `add` |
| 6 | `cmp` |
| 7 | `sub` |

如果只有二进制，也可以利用程序提供的自定义代码入口，以短输入观察栈与分支行为，自动枚举 8 个 vtable 槽位的排列。官方预期方案是调整 vtable、去掉 `matched_inst > 1` 的退出分支，并扩大 `read` 长度。

实际上不必修改二进制。按源码写一个解释器更便于检查每条语义：

- `psh` 和 `jmp` 是双字节指令，其余为单字节；
- 栈向低地址增长；
- 算术结果按 `int8_t` 截断后存入 `uint8_t` 栈；
- `cmp` 计算次栈顶减栈顶，并设置等于、大于、小于三个标志位；
- `jmp` 的低 3 位是条件掩码，高 5 位是带符号相对偏移；
- 所有置位指令按上表顺序在同一个 IP 上依次执行。

完整模拟器如下：

```python
from pathlib import Path

code = Path("code").read_bytes()
stack = [0] * 0x10000

ip = 0
sp = 0xff00
flags = 0

operations = [
    "div",
    "psh",
    "mul",
    "jmp",
    "pop",
    "add",
    "cmp",
    "sub",
]


def signed8(value):
    value &= 0xff
    return value - 256 if value & 0x80 else value


while ip < len(code) and code[ip] != 0:
    current = code[ip]

    for bit, operation in enumerate(operations):
        if not current & (1 << bit):
            continue

        if operation == "psh":
            sp -= 1
            stack[sp] = code[ip + 1]

        elif operation == "pop":
            sp += 1

        elif operation in {"add", "sub", "mul", "div"}:
            left = stack[sp + 1]
            right = stack[sp]

            if operation == "add":
                result = left + right
            elif operation == "sub":
                result = left - right
            elif operation == "mul":
                result = left * right
            else:
                # 两个 uint8_t 在 C 中先提升为非负 int。
                result = int(left / right)

            sp += 1
            stack[sp] = signed8(result) & 0xff

        elif operation == "cmp":
            difference = signed8(
                stack[sp + 1] - stack[sp]
            )

            if difference == 0:
                flags = 1
            elif difference > 0:
                flags = 2
            else:
                flags = 4

        elif operation == "jmp":
            parameter = code[ip + 1]
            if parameter & flags:
                ip += signed8(parameter) >> 3

    psh_bit = 1 << operations.index("psh")
    jmp_bit = 1 << operations.index("jmp")
    ip += 2 if current & (psh_bit | jmp_bit) else 1


raw = bytes(stack[sp:]).split(b"\x00", 1)[0]

# 原程序在输出前调用 revstr。
print(raw[::-1].decode())
```

模拟执行 215 个指令字节后，栈中形成 46 字节的反向字符串。应用程序最后的 `revstr` 语义后得到：

```text
L3AK{7h3y_70ld_m3_574ck_m4ch1n35_4r3_u53l355!}
```

## 方法总结

- 位掩码型 opcode 允许一字节同时命中多个处理函数；遇到“压缩指令”提示时，应检查解释循环是否逐位分派。
- 调试版与发布版的 opcode 顺序不同，不能仅凭枚举名称给字节贴标签。应从 vtable、测试入口或源码条件编译分支恢复真实映射。
- 官方的二进制补丁方案可行，但解释器更容易复验边界条件、带符号跳转和栈变化，也不会受输入长度限制。
- 模拟 C 代码时要保留 `uint8_t` 存储、`int8_t` 截断和整数提升语义，否则分支标志与最终字符会出现偏差。
