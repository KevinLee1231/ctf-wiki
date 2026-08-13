# Adversarial Beaver

## 题目简述

服务让用户定义不超过 100 个状态的二进制图灵机。机器的 tape 位于全局 `Machine` 对象中，磁头初始在 `tape[0]`；向左移动时，`tape_byte_index` 可以减为负数，使状态机读写 tape 之前的 `.bss` 数据。二进制未启用完整 RELRO，目标是在有限状态数内定位一个 GOT 指针，并把它调整到 libc 的 one-gadget。

## 解题过程

直接为每个向左移动的 bit 建一个状态会很快耗尽 100 状态预算。利用时改为扫描内存内容：ASLR 会改变映射基址，但函数地址的页内低 12 位不变。官方题目中目标 GOT 条目的页内模式是：

```text
GOT_ENTRY & 0xfff = 0x6f0
```

状态机先不断向左，每次跨过四个 padding bit 后比较连续 12 bit；匹配失败就回到扫描环，匹配成功便进入加法器。因为磁头向左读取，构造比较状态时需要反转 12 bit 模式的次序。

定位成功后，不必写入完整的随机化地址，只要对现有 GOT 指针做常量二进制加法。官方 solver 选用 libc 偏移 `0xebd3f` 的 one-gadget，并加入：

$$
\Delta = 0xebd3f - 0x606f0 = 0x8b64f.
$$

加法器为每一位建立“有进位”和“无进位”两个状态，根据当前 tape bit 与常量对应位写回和位，并把进位传给下一位。关键结构可以抽象为：

```python
def addition(value):
    # 从最低位递归构造两套状态：carry / no_carry
    # 每个状态根据 tape 当前位写回加法结果并向右移动
    return first_no_carry_state


program = chain(
    move_left(),
    find_bit_pattern(
        pad_length=4,
        bit_length=12,
        bit_pattern=0x6F0,
        success=chain(move_right(), addition(0x8B64F)),
    ),
)
```

官方生成器实际输出 69 个状态，满足上限。图灵机运行结束后，`printf` 等后续 GOT 间接调用会落到 one-gadget，从而取得 shell。读取 flag 得到：

```text
grey{congrats_on_solving_BB-100!}
```

## 方法总结

- 核心技巧：用有限状态机按稳定的页内地址模式扫描越界内存，再原地对随机化函数指针做常量加法。
- 识别信号：可向对象前方无限移动的 tape 加上全局存储，本质上就是受状态预算约束的 bit 级越界读写。
- 复用要点：状态紧张时不要把绝对移动距离硬编码为线性状态链；内容扫描和逐位加法能把复杂度降到与模式宽度、常量位数相关。
