# UIUCTF 2024 Time Travel

## 题目简述

程序把命令行第一个参数作为候选 flag，交给一台自定义的多时间线 Brainfuck 虚拟机。正确输入会输出 50 个点和 `Good`；错误输入则不能到达同样结果。虚拟机程序存储在二进制的 `program` 字节数组中，直接运行一次正确输入约需 20 秒，而逐字符爆破会被大量线程、等待和回滚操作拖慢。

关键不是利用线程竞态，而是逆向自定义指令集，从高度重复的 5DBF 程序中抽取每个输入字节上的确定性变换，再交给 Z3 求解。

## 解题过程

二进制用数组下标选择指令处理函数。按 `instructions[]` 的顺序，字节码可以映射回：

```text
0  nop                 >  指针右移
<  指针左移            +  当前单元加一
-  当前单元减一        ,  读取输入
.  输出单元            [ ] Brainfuck 循环
~  回滚最近一步写操作  ( ) 创建/结束子时间线
v  把指针移到下方时间线
^  把指针移到上方时间线
@  等待下方            $  等待当前时间线取得指针
%  等待上方            *  停止记录新历史
?  调试状态            !  调试退出
```

`(` 会复制当前时间线的指令指针、整条纸带、历史链和内存指针链，并启动新线程；父时间线跳过括号块，子时间线从块内继续。`v`、`^` 在相邻时间线之间移动内存指针，三个等待指令用来保证各阶段按预期交接。`~` 则根据历史链恢复最近一个时间步修改前的单元值。

先从二进制提取 `program` 数组，并按指令表还原文本：

```python
OPS = "0><+-,.[]~()v^@$%*?!"
program = "".join(OPS[byte] for byte in program_bytes)
```

还原后的程序很长，但结构高度重复。读取每个字符的时间线都包含同一个模板：先执行 `,` 读取一个字节，再加上固定的 `3` 和模板中连续 `+` 的数量。因此可为 50 次读取建立 8 位符号量：

```python
flag = [z3.BitVec(f"x{i}", 8) for i in range(50)]
chars = []

for i, match in enumerate(re.findall(INPUT_TEMPLATE, program)):
    chars.append(flag[i] + len(match) + 3)
```

后续每个变换块由八个“基数”子程序和一个偏移子程序组成。用一个普通的单时间线 Brainfuck 解释器运行这些不涉及并行指令的小片段，就能得到八个 8 位常量 `base[0..7]` 和 `offset`。一个输入字节 `x` 的变换等价于：

$$
T(x)=\mathit{offset}+\sum_{i=0}^{7}\operatorname{bit}_i(x)\cdot \mathit{base}_i\pmod {256}.
$$

也就是说，第 $i$ 位为 1 时就把对应基数加进结果。代码可以直接写成：

```python
def transform(x, base, offset):
    result = sum(
        z3.If(x & (1 << i) == (1 << i),
              z3.BitVecVal(base[i], 8),
              z3.BitVecVal(0, 8))
        for i in range(8)
    )
    return result + offset
```

某些基数片段不会终止；这表示对应输入位不能取 1，否则真实程序也会卡死。求解器需要额外加入：

```python
solver.add(chars[index] & (1 << bit) == 0)
```

各时间线按相反方向汇合，因此应逆序处理抽出的变换块，并用 `i % 50` 把它们分配回 50 个字符。题目最终要求所有变换结果都归零：

```python
for index, block in reversed(list(enumerate(transform_blocks))):
    base, offset, forbidden_bits = evaluate_block(block)
    for bit in forbidden_bits:
        solver.add(chars[index % 50] & (1 << bit) == 0)
    chars[index % 50] = transform(chars[index % 50], base, offset)

for value in chars:
    solver.add(value == 0)

assert solver.check() == z3.sat
answer = bytes(solver.model().eval(x).as_long() for x in flag)
print(answer)
```

求解结果为：

```text
uiuctf{_p4R4ll3l_c0MPut1Ng_w1tH_5dbfwmtt}\xff\xff\xff\xff\xff\xff\xff\xff\xff
```

后九个 `0xff` 不是 flag。`read_char()` 在读到 C 字符串末尾的 NUL 后返回 `-1`，以 8 位字节观察就是 `0xff`；程序仍按 50 个输入位置构造了约束。应在第一个 `}` 处截断。将得到的 41 字节候选交给发布二进制，已验证输出：

```text
..................................................Good
```

所以 flag 是：

```text
uiuctf{_p4R4ll3l_c0MPut1Ng_w1tH_5dbfwmtt}
```

## 方法总结

本题用多线程、时间线复制、指针转移和历史回滚制造了执行复杂度，但实际校验程序由重复模板生成。逆向时应先恢复虚拟机语义，再把并行调度结构压缩成每个字符上的 8 位线性选择和模 $256$ 加法，而不是尝试完整模拟所有线程交错。

Z3 只负责解最后的位约束；八个基数、偏移、变换顺序以及不终止位都必须先从字节码结构中恢复。末尾的 `0xff` 也说明建模结果仍要回到 C 运行时语义中解释，不能把模型输出机械地全部当成 flag。
