# Visual Hacker

## 题目简述

题目模拟一种非接触式密码输入系统：用户面对摄像头，用视线在虚拟平面上书写字符。附件给出按字符顺序分成 10 组的 4048 张人脸关键帧、一个训练好的凝视估计模型和一段 Ook! 消息，要求恢复 10 个字符。

Ook! 程序无需输入，翻译为 Brainfuck 后直接运行即可得到完整提示：关键帧虽然杂乱，但覆盖了书写路径的大部分点；应先用随附模型估计每帧的视线角度，再求视线与假想平面的交点分布。外部技术背景可参考 [L2CS-Net 官方实现](https://github.com/Ahmednull/L2CS-Net)，其核心是分别预测 yaw 与 pitch。

## 解题过程

### 从 90 档分类结果还原连续角度

题目源码中的 `L2CS` 以 ResNet-18 为骨干，经过全连接解码器后分别输出 90 维 yaw、pitch logits。第 $i$ 档代表角度 $4i-180$ 度，因此先做 Softmax，再计算期望：

$$
\theta=\sum_{i=0}^{89}\operatorname{softmax}(z)_i\cdot i\cdot4-180.
$$

注意 `model.py` 的返回顺序是 `yaw, pitch`，不要仅按变量名猜测。加载权重后还必须调用 `model.eval()`，否则 ResNet 中的 BatchNorm 会继续使用训练态统计量。

```python
import numpy as np
import torch
from torch import nn

from model import L2CS
from transform import transform

bins = torch.arange(90, dtype=torch.float32)
softmax = nn.Softmax(dim=1)

def decode_angles(images, model):
    batch = torch.stack([transform(image.convert("RGB")) for image in images])
    with torch.no_grad():
        yaw_logits, pitch_logits = model(batch)
        yaw_deg = (softmax(yaw_logits) * bins).sum(dim=1) * 4 - 180
        pitch_deg = (softmax(pitch_logits) * bins).sum(dim=1) * 4 - 180
    return np.deg2rad(pitch_deg.numpy()), np.deg2rad(yaw_deg.numpy())
```

`checkpoint.z01` 与 `checkpoint.zip` 是同一个分卷 ZIP，使用支持分卷的解压工具打开 `checkpoint.zip`，可得到 `checkpoint.pkl`。随后按源码结构加载：

```python
model = L2CS()
model.load_state_dict(torch.load("checkpoint.pkl", map_location="cpu"))
model.eval()
```

### 将视线投影到书写平面

以摄像机为原点，取前方平面 $z=1$。由 pitch $p$ 和 yaw $y$ 构造方向向量

$$
\boldsymbol d=(\cos p\sin y,\ \sin p,\ \cos p\cos y).
$$

射线为 $\boldsymbol r(t)=t\boldsymbol d$。令其 $z$ 坐标等于 1，可得

$$
t=\frac{1}{d_z},
\qquad
(x_s,y_s)=(-td_x,td_y).
$$

横坐标取负是把摄像机画面还原为书写者视角。不同绘图库的坐标方向不同；若字符左右镜像或旋转，只需统一交换坐标轴或翻转符号，不能逐字符任意调整。

```python
from pathlib import Path

import matplotlib.pyplot as plt
import numpy as np
from PIL import Image

def intersections(pitch, yaw):
    direction = np.column_stack((
        np.cos(pitch) * np.sin(yaw),
        np.sin(pitch),
        np.cos(pitch) * np.cos(yaw),
    ))
    scale = 1.0 / direction[:, 2]
    return np.column_stack((-scale * direction[:, 0],
                             scale * direction[:, 1]))

for group in range(10):
    paths = sorted(
        Path("Captures", str(group)).glob("*.png"),
        key=lambda path: int(path.stem),
    )
    points = []
    for start in range(0, len(paths), 16):
        images = [Image.open(path) for path in paths[start:start + 16]]
        pitch, yaw = decode_angles(images, model)
        points.append(intersections(pitch, yaw))

    points = np.concatenate(points)
    # 以书写者视角显示；散点比连线更能容忍丢帧和离群点。
    plt.scatter(points[:, 1], -points[:, 0], s=5)
    plt.axis("equal")
    plt.savefig(f"trajectory-{group}.png", dpi=200)
    plt.close()
```

逐组查看 10 张轨迹图，可辨认为：

```text
0  e  j  +  #  K  @  V  3  x
```

因此 flag 为：

```text
moectf{0ej+#K@V3x}
```

## 方法总结

本题把分类式角度回归、三维方向向量和射线—平面求交串成一条完整的数据恢复链。容易出错的地方有三个：确认模型输出头的顺序、在推理前切换到 `eval()`、统一摄像机与书写者的坐标系。最终轨迹包含噪声时，应依据整组几何形状识别字符，而不是期待每帧都落在理想笔画上。
