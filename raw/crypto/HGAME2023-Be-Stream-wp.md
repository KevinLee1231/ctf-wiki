# Be Stream

## 题目简述

程序用一个二阶线性递推序列产生异或密钥流，但直接递归计算的下标增长到 $(i//2)^6$，朴素实现几乎无法运行。递推关系为：

$$
S_i=4S_{i-1}+7S_{i-2}\pmod{256}
$$

可以用 $2\times2$ 矩阵快速幂在 $O(\log n)$ 时间求任意项，再与密文字节异或恢复 flag。

## 解题过程

初值来自字符串 `Be water` 和 `my friend` 转成的大整数；实际运算始终模 256，所以只保留最低字节。递推可写成：

$$
\begin{aligned}
\begin{bmatrix}S_i\\S_{i-1}\end{bmatrix}
&=
\begin{bmatrix}4&7\\1&0\end{bmatrix}
\begin{bmatrix}S_{i-1}\\S_{i-2}\end{bmatrix}
\pmod{256}
\end{aligned}
$$

下面是无需 SageMath 的完整实现：

```python
enc = (
    b"\x1a\x15\x05\t\x17\t\xf5\xa2-\x06\xec\xed\x01-\xc7\xcc"
    b"2\x1eXA\x1c\x157[\x06\x13/!-\x0b\xd4\x91-\x06\x8b\xd4"
    b"-\x1e+*\x15-pm\x1f\x17\x1bY"
)
key = [
    int.from_bytes(b"Be water", "big"),
    int.from_bytes(b"my friend", "big"),
]


def matrix_mul(left, right):
    result = [[0, 0], [0, 0]]
    for row in range(2):
        for column in range(2):
            for index in range(2):
                result[row][column] += left[row][index] * right[index][column]
            result[row][column] %= 256
    return result


def stream_at(index):
    if index == 0:
        return key[0] % 256
    if index == 1:
        return key[1] % 256

    result = [[1, 0], [0, 1]]
    base = [[4, 7], [1, 0]]
    exponent = index

    while exponent:
        if exponent & 1:
            result = matrix_mul(base, result)
        base = matrix_mul(base, base)
        exponent >>= 1

    return (result[1][0] * key[1] + result[1][1] * key[0]) % 256


flag = bytes(
    byte ^ stream_at((index // 2) ** 6)
    for index, byte in enumerate(enc)
)
print(flag.decode())
```

运行得到：

```text
hgame{1f_this_ch@l|eng3_take_y0u_to0_long_time?}
```

由于所有项都模 256，这个具体序列还具有 64 的周期；也可以先算出一个周期，再用下标模 64 查表。但矩阵快速幂不依赖预先发现周期，适用范围更广。

## 方法总结

- 核心技巧：把带常系数的二阶递推转换为矩阵乘法，并用二进制快速幂求超大下标。
- 易错点：每次矩阵运算都要模 256；两个初值的顺序和题目索引 `(i // 2) ** 6` 都不能颠倒。
- 复用要点：递归深、指数大但状态维度固定时，应优先寻找线性递推、矩阵快速幂或模意义下的周期。
