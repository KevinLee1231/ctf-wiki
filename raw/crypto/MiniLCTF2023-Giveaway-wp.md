# MiniLCTF2023 - Giveaway

## 题目简述

服务把 flag 所在的 512 位向量 `dough` 与随机 512 位向量 `chocolate_jam` 在 $GF(2)$ 上做内积：

$$
cookie=\langle dough,chocolate\_jam\rangle \pmod 2.
$$

每次购买曲奇会返回一行 `(cookie, chocolate_jam)`，单次连接可取得 503 至 508 组样本。决定性障碍不是哈希或随机数预测，而是在秩略有不足的二元线性方程组中恢复满足 flag 格式的解。

## 解题过程

将每个 `chocolate_jam` 按源码的高位在前顺序展开为 512 位行向量，所有返回值组成

$$
Mx=b,
$$

其中 $x$ 就是 `dough`。在 $GF(2)$ 上求一个特解 $x_0$ 和右零空间基 $v_1,\ldots,v_r$，全部解为

$$
x=x_0+\sum_{i=1}^{r}\lambda_i v_i,\qquad \lambda_i\in\{0,1\}.
$$

题目通常只缺 4 至 9 个秩，因此最多枚举 $2^9$ 个组合。下面代码是收集完 `rows` 和 `values` 后的核心恢复逻辑；不能直接写成 `kernel[i]`，因为需要枚举的是零空间基向量的所有线性组合。

```python
from Crypto.Util.number import long_to_bytes
from sage.all import GF, Matrix, vector


def bits_msb(value, width):
    return [int(x) for x in f"{value:0{width}b}"]


M = Matrix(GF(2), [bits_msb(jam, 512) for jam in rows])
b = vector(GF(2), values)

particular = M.solve_right(b)
basis = M.right_kernel().basis()
print(f"rank={M.rank()}, kernel_dimension={len(basis)}")

for mask in range(1 << len(basis)):
    candidate = particular
    for i, v in enumerate(basis):
        if mask >> i & 1:
            candidate += v
    raw = long_to_bytes(int("".join(map(str, candidate)), 2))
    if raw.startswith(b"miniL{"):
        print(raw)
```

仓库中的 `secret.py` 可用于核对结果：`miniL{We1c0me_TO_M1niL2o23_Crypt0!} Enjoy the challenges ahead! `。服务端在不同连接之间错误地复用了同一个 `message`，所以也可以跨两次连接收集超过 512 行后直接求解；这属于非预期简化，不改变线性模型。

## 方法总结

看到 XOR 后再求和的“点积”应首先确认运算域；本题的 `and` 与 `xor` 正好对应 $GF(2)$ 上的乘法与加法。方程不足时不要因矩阵不可逆就停止，而应求特解与零空间，再用已知 flag 前缀筛选仿射解空间。实现时还必须保持位序与服务端 `dec2blist` 一致。
