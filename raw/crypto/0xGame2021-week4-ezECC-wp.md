# week4ezECC

## 题目简述

题目在有限域椭圆曲线上给出基点 $G$、公钥 $K=kG$，以及加密点

$$
C_1=P+rK,qquad C_2=rG.
$$

曲线规模较小，可以求离散对数 $k$，再恢复编码了 flag 的点 $P$。

## 解题过程

由 $K=kG$ 求出私钥 $k$ 后，有

$$
kC_2=krG=rK,
$$

因此

$$
P=C_1-kC_2.
$$

源码把 flag 花括号内的 8 个小写字母分成两组，每个字母按 `a=01`、$\ldots$、`z=26` 编成两位十进制数；前四个字母组成 $P$ 的横坐标，后四个组成纵坐标。

```python
# SageMath
p = 14050339
E = EllipticCurve(GF(p), [1, 3243167])

G = E(7112688, 7410262)
K = E(6562993, 2753874)
C1 = E(3095063, 1465594)
C2 = E(6437074, 4385056)

k = discrete_log(K, G, operation="+")
P = C1 - k * C2

alphabet = "abcdefghijklmnopqrstuvwxyz"

def decode_coordinate(value):
    digits = f"{int(value):08d}"
    indexes = [int(digits[i:i + 2]) for i in range(0, 8, 2)]
    if any(index < 1 or index > 26 for index in indexes):
        raise ValueError("坐标不符合两位字母序号编码")
    return "".join(alphabet[index - 1] for index in indexes)

inner = decode_coordinate(P[0]) + decode_coordinate(P[1])
print(k)
print(f"0xGame{{{inner}}}")
```

得到：

```text
k = 4282465
0xGame{learnecc}
```

## 方法总结

这是简化的椭圆曲线 ElGamal 结构：私钥满足 $K=kG$，解密为 $P=C_1-kC_2$。小曲线上的离散对数可直接计算；恢复点后还要严格按源码的定长十进制字母序号解码。
