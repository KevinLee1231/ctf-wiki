# 智械危机(#1)

## 题目简述

附件给出一个仅含单层 Dense 的 Keras 模型和它对隐藏 128 维向量产生的 64 维输出。模型参数全部可读，关系为 $y=xW+b$。虽然方程数少于未知数，但服务端只要求提交向量与真实向量的平均绝对误差小于 $0.18$，因此可以把输入限制在 $[0,1]$ 并用梯度下降寻找可接受近似。

## 解题过程

模型摘要为：

```text
Input:  (None, 128)
Dense:  (None, 64)
Params: 8256 = 128 * 64 + 64
```

Dense 层没有额外非线性时：

$$
y=xW+b.
$$

从模型中直接取出 $W$ 和 $b$，把待优化变量写成 $x=\operatorname{sigmoid}(z)$，就能天然把每一维限制在 $0$ 到 $1$。损失函数应与服务端一致，使用平均绝对误差而不是文件中函数名误写的 “mse”：

```python
import numpy as np
import tensorflow as tf
from tensorflow.keras.models import load_model

model = load_model("model.h5")
W, b = model.layers[-1].get_weights()
target = np.loadtxt("encrypted.txt", dtype=np.float32).reshape(1, 64)

W = tf.constant(W, dtype=tf.float32)
b = tf.constant(b, dtype=tf.float32)
target = tf.constant(target, dtype=tf.float32)

z = tf.Variable(tf.zeros((1, 128), dtype=tf.float32))
optimizer = tf.keras.optimizers.Adam(learning_rate=1e-3)

for step in range(10000):
    with tf.GradientTape() as tape:
        x = tf.sigmoid(z)
        prediction = tf.matmul(x, W) + b
        loss = tf.reduce_mean(tf.abs(prediction - target))

    gradient = tape.gradient(loss, z)
    optimizer.apply_gradients([(gradient, z)])

    if step % 500 == 0:
        print(step, float(loss))

candidate = tf.sigmoid(z).numpy().reshape(-1)
binary = (candidate >= 0.5).astype(np.float32)
print(" ".join(map(str, binary)))
```

把 128 个数用空格分隔后提交。服务端的实际判定是：

```python
threshold = 0.18

def score(true, predict):
    return np.average(np.abs(true - predict))
```

误差低于阈值后，服务端返回：

```text
hgame{@1tCh479vCYUQI3epIXU7TQ99e^ZuEKz}
```

如果二值化使误差反而升高，也可以直接提交连续的 sigmoid 输出；应以本地重算的模型输出误差和服务端输入格式为准。

## 方法总结

- 核心技巧：白盒读取 Dense 权重，把模型输出反演成满足约束的输入向量。
- 关键认识：$64$ 个方程不足以唯一确定 $128$ 个变量，但宽松的 MAE 阈值只要求找到一个足够接近真实向量的可行解。
- 分类依据：决定性步骤是模型参数提取与梯度优化，而不是普通矩阵逆运算或程序逆向，因此归入 AI/ML。
