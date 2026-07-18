# QQQRcode

## 题目简述

服务端要求提交一个 $21\times21\times21$ 的布尔立方体，并限制其中 `1` 的数量小于 $390$。它分别沿三个坐标轴做逻辑或投影，将三个 $21\times21$ 平面当作二维码解码，结果必须依次为 `Azure`、`Assassin` 和 `Alliance`。

题目的本质是稀疏三维重建：用尽量少的体素同时覆盖三张二维码的黑色模块。一个体素最多能为三个投影各贡献一个黑点，因此应优先选择三张投影黑点的交会位置，再删除冗余体素。

## 解题过程

### 还原输入与投影规则

服务端要求输入长度恰好为 $21^3=9261$，只允许字符 `0` 和 `1`，且：

```python
input_data.count("1") < 390
```

解析顺序是外层 `z`、中层 `y`、内层 `x`：

```python
for z in range(21):
    for y in range(21):
        for x in range(21):
            data[x][y][z] = input_str[index] == "1"
```

三个投影分别为：

$$
F(x,y)=\bigvee_z D(x,y,z),\qquad
L(y,z)=\bigvee_x D(x,y,z),\qquad
T(x,z)=\bigvee_y D(x,y,z).
$$

服务端将 $F$、$L$、$T$ 依次解码为 `Azure`、`Assassin`、`Alliance`。由于投影使用逻辑或，一个位置只要存在至少一个体素就是黑色；同一直线上放置多个体素不会改变该投影，因而会产生可删除的冗余。

### 生成三张目标二维码

三个字符串都能放入版本 1 的二维码，即 $21\times21$ 个模块。生成时固定低纠错级别、无静区，保证矩阵尺寸与服务端一致：

```python
import qrcode
import numpy as np

def qr_matrix(text):
    qr = qrcode.QRCode(
        version=1,
        error_correction=qrcode.constants.ERROR_CORRECT_L,
        box_size=1,
        border=0,
    )
    qr.add_data(text)
    qr.make(fit=False)
    return np.array(qr.get_matrix(), dtype=np.uint8)

front = qr_matrix("Azure")
left  = qr_matrix("Assassin")
top   = qr_matrix("Alliance")
```

### 取三重交会并贪心删点

若 $F(x,y)$、$L(y,z)$、$T(x,z)$ 都是黑点，则体素 $(x,y,z)$ 可以同时覆盖三张投影。先只保留这些三重交会：

```python
box = (
    front[:, :, None]
    + left[:, None, :]
    + top[None, :, :]
    == 3
).astype(np.uint8)
```

随后维护每条投影射线上的体素数量。对候选体素 $(x,y,z)$，只有三条对应射线的计数都大于 1 时才删除它，这样删除后每个当前已覆盖的投影点仍至少有一个体素：

```python
xoy = box.sum(axis=2)
xoz = box.sum(axis=1)
yoz = box.sum(axis=0)

for x, y, z in candidates:
    if xoy[x, y] > 1 and xoz[x, z] > 1 and yoz[y, z] > 1:
        box[x, y, z] = 0
        xoy[x, y] -= 1
        xoz[x, z] -= 1
        yoz[y, z] -= 1
```

随机打乱 `candidates` 后多运行几次，可以得到不同的局部最优解。题目仓库的生成器还强制加入 `box[10][7][10]` 作为经验修正。只取三重交会时，某些只在一张或两张目标二维码中为黑色的模块无法被精确覆盖；额外体素会让投影与理想矩阵存在少量差异，但二维码纠错和解码器的镜像处理仍可恢复三个字符串。最终解中 `1` 的数量可以压到 300 以下，满足 $390$ 的限制。

### 按服务端顺序序列化

最稳妥的方式是严格照服务端的 `z → y → x` 顺序输出：

```python
payload = "".join(
    str(int(box[x, y, z]))
    for z in range(21)
    for y in range(21)
    for x in range(21)
)
assert len(payload) == 21 ** 3
assert payload.count("1") < 390
```

仓库原生成器使用的是 `z → x → y`，相当于把每个切片转置；二维码解码器能识别镜像矩阵，所以该版本同样可用，但自行实现时不应无意混用两种坐标顺序。

连接服务后先完成四字符 SHA-256 工作量证明，再提交整段 `payload`。三个投影均解码成功后，服务端输出 flag。

## 方法总结

构造的核心是“一个体素尽可能同时服务三个投影”：先取三重黑点交会，再根据三条射线的覆盖计数贪心删去冗余。二维码纠错允许最终投影存在少量误差，因此不必求解严格的三维离散层析最优问题。实现中最容易出错的是坐标含义、数组轴顺序和序列化顺序，提交前应在本地用与服务端完全相同的投影及解码代码做一次端到端验证。
