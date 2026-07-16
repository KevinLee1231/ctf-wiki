# RNG

## 题目简述

题目自行实现了 MT19937，以 32 位随机 seed 初始化 624 个状态字，随后连续输出一整轮共 624 个随机数。服务要求提交最初的 seed。

624 个输出足以逐个逆转 temper 得到 twist 后的完整状态。再按照源码的原地更新顺序逆转 twist，恢复初始化状态中的 `mt[1]`，最后逆转初始化递推即可求出 seed；仅克隆后续输出并不能直接完成本题。

## 解题过程

输出前的 temper 为：

```python
y = y ^ y >> 11
y = y ^ y << 7 & 0x9D2C5680
y = y ^ y << 15 & 0xEFC60000
y = y ^ y >> 18
```

右移异或和带掩码左移异或都可以用固定点迭代逆转。对 624 个输出分别 untemper 后，得到 twist 后的数组 `new`。

源码的 twist 是原地执行的。倒序恢复第 $i$ 项时：

- 若 $i\geq227$，式中的 `mt[(i+397)%624]` 已在正向过程中更新，应取 `new[i-227]`；
- 若 $i<227$，该引用仍是旧状态，并且倒序时 `old[i+397]` 已经恢复。

最后一次正向迭代会使用已更新的 `mt[0]`，所以旧 `mt[0]` 的低 31 位不会直接恢复。不过 `old[1]` 是完整的，而初始化满足

$$
mt[1]=1812433253\cdot(seed\oplus(seed\gg30))+1\pmod{2^{32}}.
$$

常数 1812433253 为奇数，在模 $2^{32}$ 下存在乘法逆元，因此可由 `mt[1]` 反推出 seed。

完整脚本如下：

```python
import ast

from pwn import remote


HOST = "TARGET"
PORT = 10006

MASK = 0xFFFFFFFF
N = 624
A = 0x9908B0DF
UPPER = 0x80000000
LOWER = 0x7FFFFFFF


def undo_right_xor(value, shift):
    result = value
    for _ in range(32 // shift + 2):
        result = value ^ (result >> shift)
    return result & MASK


def undo_left_xor_mask(value, shift, mask):
    result = value
    for _ in range(32 // shift + 2):
        result = value ^ ((result << shift) & mask)
    return result & MASK


def untemper(value):
    value = undo_right_xor(value, 18)
    value = undo_left_xor_mask(value, 15, 0xEFC60000)
    value = undo_left_xor_mask(value, 7, 0x9D2C5680)
    value = undo_right_xor(value, 11)
    return value


def untwist(new):
    old = [0] * N

    for i in range(N - 1, -1, -1):
        if i >= 227:
            reference = new[i - 227]
        else:
            reference = old[i + 397]

        value = (new[i] ^ reference) & MASK

        # value 的最高位就是 twist 前拼接值 y 的最低位。
        low_bit = value >> 31
        if low_bit:
            value ^= A
        y = ((value << 1) & MASK) | low_bit

        old[i] = (old[i] & LOWER) | (y & UPPER)

        # i=623 时低 31 位来自已经更新的 new[0]，不是 old[0]。
        if i < N - 1:
            old[i + 1] = (old[i + 1] & UPPER) | (y & LOWER)
        else:
            assert (y & LOWER) == (new[0] & LOWER)

    return old


def recover_seed(outputs):
    post_twist = [untemper(value) for value in outputs]
    initialized = untwist(post_twist)

    multiplier = 1812433253
    inverse_multiplier = pow(multiplier, -1, 1 << 32)
    mixed_seed = ((initialized[1] - 1) * inverse_multiplier) & MASK
    return undo_right_xor(mixed_seed, 30)


io = remote(HOST, PORT)
io.recvuntil(b"[+] result:\n")
outputs = ast.literal_eval(io.recvline().decode())
assert len(outputs) == 624

seed = recover_seed(outputs)
io.sendlineafter(b">", str(seed).encode())
print(io.recvline().decode().strip())
```

提交恢复出的 seed 后得到：

```text
0xGame{2569bd55-a14d-46d8-81f5-e1397e4be7bc}
```

## 方法总结

MT19937 的 624 个完整输出会泄露一整组内部状态，但本题需要的是初始化 seed，因此还必须逆转状态转换。处理原地 twist 时要区分循环前半段引用的旧状态与后半段引用的新状态；即使旧 `mt[0]` 不能全部直接恢复，也可以利用完整的 `mt[1]` 和可逆初始化公式得到 seed。
