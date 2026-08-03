# CPSC

## 题目简述

题目给出一个名为 `cpsc` 的原生程序：输入密码后，程序把输入原地“混合”为十六进制字符串，并与内置的 43 字节目标比较。二进制由 CPC 编译器生成；CPC 会把带 `cps` 标记的 C 函数转换为 continuation-passing style，普通循环和递归在反编译结果中会呈现为大量状态、continuation 和间接控制流。

识别 CPC 的线索包括残留的断言字符串、运行库符号以及异常的控制流形态。用同一 CPC 工具链编译一个小循环或递归示例，可以确认这些状态函数只是源级循环的展开。恢复高层语义后，真正的算法由三部分组成：递归二分、交错合并，以及两个方向的字节级 xorshift 累积变换；整个过程不改变长度。

## 解题过程

### 还原正向算法

设当前递归节点编号为 $n$。`mix` 把字符串从中点分为左右两段，分别以子节点编号 $2n$、$2n+1$ 递归处理，再执行：

1. 按“右一字节、左一字节”的顺序交错合并；
2. 把合并结果整体反转；
3. 以 $47n$ 为初值从左到右做一次 XOR 累积与 xorshift；
4. 以 $51n$ 为初值从右到左再做一次。

单字节 xorshift 为：

```c
c ^= c << 6;
c ^= c >> 7;
c ^= c << 1;
```

所有运算截断到 8 位。内置目标为：

```text
e338e9cc0199e8c24b43760f2277cf56f9b7ddff343aaf116fe26cafca4538cfb9c26477e377d19a301e13
```

### 逐层构造逆变换

对 $y=x\oplus(x\ll s)$，可以从低位到高位恢复 $x$；对 $y=x\oplus(x\gg s)$，则从高位向低位恢复。xorshift 的三步必须按相反顺序撤销：先撤销左移 1，再撤销右移 7，最后撤销左移 6。

一次累积 pass 的正向关系为：

$$
o_i=T(p_i\oplus o_{i-1}),
$$

其中 $T$ 是 xorshift，$o_{-1}$ 是种子。因此：

$$
p_i=T^{-1}(o_i)\oplus o_{i-1}.
$$

先撤销反向的 $51n$ pass，再撤销正向的 $47n$ pass；随后整体反转，把偶数位置还给右半段、奇数位置还给左半段，最后按递归树逆向处理两半。核心 solver 如下：

```python
def undo_lshift_xor(y, shift):
    x = 0
    for i in range(8):
        yi = (y >> i) & 1
        dependency = (x >> (i - shift)) & 1 if i >= shift else 0
        x |= (yi ^ dependency) << i
    return x

def undo_rshift_xor(y, shift):
    x = 0
    for i in range(7, -1, -1):
        yi = (y >> i) & 1
        dependency = (x >> (i + shift)) & 1 if i + shift < 8 else 0
        x |= (yi ^ dependency) << i
    return x

def undo_xorshift(value):
    value = undo_lshift_xor(value, 1)
    value = undo_rshift_xor(value, 7)
    value = undo_lshift_xor(value, 6)
    return value

def undo_pass(data, seed):
    previous = seed & 0xff
    out = bytearray()
    for encoded in data:
        out.append(undo_xorshift(encoded) ^ previous)
        previous = encoded
    return out

def undo_xor(data, node):
    data = bytearray(reversed(data))
    data = undo_pass(data, 51 * node)
    data.reverse()
    return undo_pass(data, 47 * node)

def unmerge(data, node):
    data = undo_xor(data, node)
    data.reverse()
    right = bytearray(data[0::2])
    left = bytearray(data[1::2])
    return left, right

def unmix(data, node=1):
    if len(data) <= 1:
        return data
    left, right = unmerge(data, node)
    return unmix(left, node << 1) + unmix(right, (node << 1) | 1)

target = bytes.fromhex(
    "e338e9cc0199e8c24b43760f2277cf56f9b7ddff343aaf11"
    "6fe26cafca4538cfb9c26477e377d19a301e13"
)
flag = unmix(bytearray(target))
print(flag.decode())
```

运行后得到：

```text
uiuctf{n41tv3_func7iona1_pr0gr4mm1ng_1n_C!}
```

将该字符串重新送入正向程序，应得到完全相同的 86 位十六进制目标；这是比“输出看起来像 flag”更可靠的最终校验。

## 方法总结

- 核心技巧：先把 CPC 生成的 continuation 控制流恢复为递归混合算法，再按“反向 pass、反转、拆交错、递归”的顺序逐层求逆。
- 识别信号：二进制包含 CPC 运行库或断言痕迹，反编译控制流极其破碎，但数据长度始终不变且最终只是固定字节比较。
- 复用要点：xorshift 的逆序不仅要反转三条语句，还要根据移位方向选择恢复位的遍历方向；带反馈的 XOR pass 则必须使用前一个密文字节，而不是前一个明文字节。
