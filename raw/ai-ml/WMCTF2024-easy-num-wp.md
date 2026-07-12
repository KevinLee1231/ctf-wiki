# easy num

## 题目简述

题目给出一个 TensorFlow/Keras 模型服务。输入为 20 个数，服务将其 reshape 成 `(1, 20, 1)` 后预测。若模型输出大于 `0.99` 且输入均值大于 `0.5`，返回 flag。

核心判断：

```python
def predict(input_data):
    processed_data = np.array(input_data).reshape(1, 20, 1)
    prediction = model.predict(processed_data)

    if prediction[0][0] > 0.99 and np.mean(input_data) > 0.5:
        return "FLAG{}"

    return f"模型预测输出: {prediction[0][0]}"
```

## 解题过程

### 关键机制

这是对抗样本题。目标不是理解模型语义，而是构造一个满足输出阈值的输入。使用 FGSM 时，对输入张量求损失函数梯度，再沿梯度符号方向扰动，并裁剪到 $[0,1]$。

FGSM 形式：

$$x'=\operatorname{clip}(x+\epsilon\cdot \operatorname{sign}(\nabla_x L(y, f(x))),0,1)$$

### 求解步骤

随机生成基础输入，多次尝试生成对抗样本：

```python
import tensorflow as tf
import numpy as np

def generate_adversarial_example(model, input_data, epsilon=0.1):
    input_tensor = tf.convert_to_tensor(input_data.reshape(1, 20, 1), dtype=tf.float32)
    with tf.GradientTape() as tape:
        tape.watch(input_tensor)
        prediction = model(input_tensor)
        true_label = tf.convert_to_tensor([[1]], dtype=tf.float32)
        loss = tf.keras.losses.binary_crossentropy(true_label, prediction)

    gradient = tape.gradient(loss, input_tensor)
    adversarial_input = input_tensor + epsilon * tf.sign(gradient)
    adversarial_input = tf.clip_by_value(adversarial_input, 0, 1)
    return adversarial_input.numpy().reshape(20)
```

检查条件并发送给服务：

```python
for _ in range(1000):
    base_input = np.random.rand(20)
    x = generate_adversarial_example(model, base_input)
    pred = model.predict(x.reshape(1, 20, 1))[0][0]
    if pred > 0.99 and np.mean(x) > 0.5:
        print(" ".join(map(str, x)))
        break
```

## 方法总结

- 阈值条件完全暴露，适合直接做白盒对抗样本。
- 均值约束要求扰动后整体偏大，可通过调大 `epsilon` 或多次随机初始点解决。
- 不需要恢复模型训练数据，只需满足服务端判定。
