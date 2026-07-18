# week4Predict 2

## 题目简述

服务端使用长度 45 的线性反馈序列：

$$
x_{t+45}\equiv\sum_{j=0}^{44}a_jx_{t+j}\pmod p,
\qquad p=1435756429.
$$

初始资金允许故意猜错 90 次取得连续输出；恢复 45 个系数后，持续正确预测使资金达到 600，即可获得 flag。

## 解题过程

收集 $x_0,\ldots,x_{89}$ 后，前 45 个递推关系组成线性方程组

$$
M\vec a=\vec v,
$$

其中

$$
M_{i,j}=x_{i+j},\quad0\le i,j<45,
\qquad
\vec v=(x_{45},\ldots,x_{89})^T.
$$

全部运算都在有限域 $\mathbb Z_p$ 上进行，故可用 SageMath 的 `solve_right` 直接求 $\vec a$。源码初始化资金为 97，90 次错误后剩 7；每次正确预测净增 1，预测 601 次足以越过 600。

```python
# SageMath
from pwn import remote

HOST = "challenge.example"
PORT = 0
MODULUS = 1435756429
io = remote(HOST, PORT)

states = []
for _ in range(90):
    io.sendlineafter(b">", b"1")
    io.sendlineafter(b">", b"1500000000")  # 大于模数，保证猜错
    io.recvuntil(b"Right Answer is ")
    states.append(int(io.recvline()))

ring = Zmod(MODULUS)
M = matrix(ring, [states[i:i + 45] for i in range(45)])
v = vector(ring, states[45:90])
mask = list(M.solve_right(v))

# 用所有已知关系核验恢复出的系数
for i in range(45):
    predicted = sum(mask[j] * states[i + j] for j in range(45))
    assert int(predicted % MODULUS) == states[i + 45]

window = states[45:90]
for _ in range(601):
    value = sum(int(mask[j]) * window[j] for j in range(45)) % MODULUS
    window = window[1:] + [value]
    io.sendlineafter(b">", b"1")
    io.sendlineafter(b">", str(value).encode())

print(io.recvrepeat(1).decode())
```

服务端动态 flag 的格式为：

```text
0xGame{87087788-7777-[三位十六进制]5-3[三位十六进制]-51284432[四位数字]}
```

原 WP 记录的 `0xGame{87087788-7777-baa5-340d-512844320391}` 是一次实际样例，不是固定答案。

## 方法总结

线性递推的未知系数可以由足够多的连续输出转化为模意义下的线性方程组。长度为 $L$ 时通常需要 $2L$ 个连续状态；求解后必须回代全部 45 条关系，再开始远程预测。
